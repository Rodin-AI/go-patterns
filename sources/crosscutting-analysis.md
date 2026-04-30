# Cross-Cutting Concerns: How Mature Codebases Handle the Hard Parts

Cross-cutting concerns are the things that touch everything
but belong nowhere. How a codebase handles logging,
telemetry, config, retry, and lifecycle management reveals
its architectural philosophy more than any feature code.

---

## 1. Logging: From Strings to Semantic Channels

### CockroachDB: Channel-Based Log Routing

CockroachDB doesn't just log at severity levels — it
routes logs to **semantic channels**:

```go
const DEV = logpb.Channel_DEV       // development noise
const OPS = logpb.Channel_OPS       // operator actions
const HEALTH = logpb.Channel_HEALTH // background health
const STORAGE = logpb.Channel_STORAGE
const SESSIONS = logpb.Channel_SESSIONS
const SQL_SCHEMA = logpb.Channel_SQL_SCHEMA
const USER_ADMIN = logpb.Channel_USER_ADMIN
```

Each channel can be routed to different sinks (file,
network, etc.) independently. Production deploys typically
disable DEV entirely and route HEALTH to monitoring.

**Force:** In a multi-tenant distributed database, "who
cares about this log?" is a different question than "how
bad is it?" An INFO-level schema change matters to DBAs
but not to SREs monitoring node health.

**Ecosystem insight:** The channel IS the audience. When
you write `log.Health.Warningf(...)`, you're declaring
"the person watching cluster health needs to see this."
Severity is orthogonal to audience.

### Prometheus: Self-Instrumentation

Prometheus instruments itself with its own metrics:

```go
type scrapeMetrics struct {
    targetScrapeSampleLimit        prometheus.Counter
    targetScrapeSampleOutOfOrder   prometheus.Counter
    targetIntervalLengthHistogram  *prometheus.HistogramVec
    // ... 20+ metrics
}
```

Metrics are collected in a struct, constructed once via
`newScrapeMetrics(reg)`, and passed to subsystems. No
global registration — the registerer is injected.

**Force:** Prometheus IS the metrics system. If it used
a different metrics library to instrument itself, that
would be a design smell. Dogfooding proves the API works.

### Ecto + Oban: Telemetry as Standard

Both use Erlang's `:telemetry` library with predictable
naming:

```elixir
# Oban
:telemetry.execute([:oban, :job, :start], measurements, meta)
:telemetry.execute([:oban, :job, :stop], measurements, meta)
:telemetry.execute([:oban, :job, :exception], measurements, meta)

# Ecto (adapter-emitted)
[:my_app, :repo, :query]
```

**Force:** The BEAM ecosystem standardized on `:telemetry`
for observability. Libraries don't own their monitoring —
they emit events; consumers attach handlers. This inverts
the logging relationship: the library doesn't decide what
to do with the information.

---

## 2. Config Propagation: Three Models

### CockroachDB: Cluster Settings (Distributed Config)

```go
settings.RegisterDurationSetting(
    settings.ApplicationLevel,
    "bulkio.ingest.flush_delay",
    "amount of time to wait before sending a file...",
    0,  // default
)
```

Settings are:
- **Typed** (Duration, Bool, Int, String)
- **Leveled** (ApplicationLevel vs SystemVisible)
- **Validated** (NonNegativeInt, etc.)
- **Distributed** (propagated across all nodes)
- **Version-gated** (new settings require cluster version)

Usage: `settings.Version.IsActive(ctx, clusterversion.V26_2)`

**Force:** In a distributed database, config isn't a file
— it's consensus. Every node must agree on every setting,
and settings can only be enabled once all nodes support
them. The version gate is the safety mechanism.

### Prometheus: ApplyConfig (Hot Reload)

```go
func (m *Manager) ApplyConfig(cfg *config.Config) error {
    m.mtxScrape.Lock()
    defer m.mtxScrape.Unlock()
    // rebuild scrape pools from new config
    // close old loggers, open new ones
}
```

Config is a struct loaded from YAML. On SIGHUP (or API
call), the entire config is re-parsed and `ApplyConfig`
is called on each subsystem. Each subsystem holds a mutex
and swaps atomically.

**Force:** Prometheus runs as a single binary. Config
reload must be atomic per-subsystem but doesn't need
distributed consensus. The mutex-per-subsystem pattern
gives independent reload without global coordination.

### Ecto + Oban: Config at Init, Validated Once

```elixir
# Oban validates exhaustively at startup
Validation.validate_schema(opts,
    engine: {:behaviour, Oban.Engine},
    queues: {:custom, &validate_queues/1},
    repo: {:module, [config: 0]},
    ...
)
```

Config is validated once at startup and stored as an
immutable struct. No hot reload. If config is wrong,
you know immediately (fail fast).

**Force:** Elixir/OTP applications restart processes to
apply new config. Hot reload is handled by supervisor
restarts, not config mutation. The "config as immutable
struct" pattern means no runtime config bugs — it either
passes validation at startup or the app doesn't start.

---

## 3. Retry and Resilience

### CockroachDB: Iterator-Based Retry

```go
opts := retry.Options{
    InitialBackoff: 100 * time.Millisecond,
    MaxBackoff:     2 * time.Second,
    Multiplier:     2,
    MaxRetries:     5,
}
for r := retry.StartWithCtx(ctx, opts); r.Next(); {
    // attempt operation
    if err == nil { break }
}
```

Retry is a **for-loop iterator**. `r.Next()` handles
backoff timing and returns false when exhausted. This
means retry logic reads like normal code — no callbacks,
no framework.

**Force:** CockroachDB has hundreds of retry sites. A
callback-based retry would create deeply nested code.
The iterator pattern keeps retry at the same indentation
level as the operation.

### Oban: Repo Dispatch with Built-In Retry

```elixir
defp dynamic_dispatch(conf, name, args, attempt) do
    with_dynamic_repo(conf, fn repo ->
        apply(repo, name, args)
    end)
rescue
    error in UndefinedFunctionError ->
        if attempt < @retry_opts[:retry] do
            jittery_sleep(attempt * @retry_opts[:delay])
            dynamic_dispatch(conf, name, args, attempt + 1)
        else
            reraise error, __STACKTRACE__
        end
end
```

Every Ecto operation dispatched through Oban's repo
wrapper gets automatic retry for transient failures.
The consumer never sees the retry — it's invisible
infrastructure.

**Key insight:** Oban retries `UndefinedFunctionError`
on the repo module itself — absorbing the window during
hot code reload when the module doesn't exist. This is
an ecosystem-level concern (BEAM hot code loading) handled
transparently.

---

## 4. Resource Lifecycle: The Stopper Pattern

### CockroachDB: Stopper as Universal Lifecycle

```go
type Stopper struct { ... }

// RunTask runs a synchronous task
func (s *Stopper) RunTask(ctx context.Context, taskName string, f func(context.Context)) error

// RunAsyncTask runs a goroutine tracked by the stopper
func (s *Stopper) RunAsyncTask(ctx context.Context, taskName string, f func(context.Context)) error

// ShouldQuiesce returns a channel closed when shutdown begins
func (s *Stopper) ShouldQuiesce() <-chan struct{}

// Stop initiates graceful shutdown
func (s *Stopper) Stop(ctx context.Context)
```

Every goroutine in CockroachDB is launched through a
Stopper. This gives:
- **Tracking**: know exactly which goroutines are running
- **Graceful shutdown**: quiesce signal before hard stop
- **Leak detection**: `PrintLeakedStoppers` in tests
- **Throttling**: semaphore limits async tasks

```go
func init() {
    leaktest.PrintLeakedStoppers = PrintLeakedStoppers
}
```

**Force:** A database cannot afford goroutine leaks —
they hold locks, connections, and file handles. The
Stopper is the universal answer: every background task
is accounted for, every shutdown is graceful, every leak
is detected in tests.

### Oban: Registry-Based Lifecycle

```elixir
children = [
    {Notifier, conf: conf, name: Registry.via(name, Notifier)},
    {Nursery, conf: conf, name: Registry.via(name, Nursery)},
    ...
]
```

OTP already provides lifecycle management via supervisors.
Oban's addition is the Registry — namespacing processes
so multiple instances can coexist. Lifecycle is delegated
to the platform; naming is the library's concern.

---

## 5. What These Patterns Teach for Code Review

### Questions to Ask About Cross-Cutting Concerns:

1. **Logging:** Who is the audience for this log? Is there
   a routing mechanism, or does everything go to stdout?
   Does the log help the *operator*, not just the developer?

2. **Config:** How does config reach this code? Is it
   validated at startup or silently wrong at runtime? Can
   it be changed without restart? Should it be?

3. **Retry:** Is retry happening at the right layer? Is it
   invisible to the caller? Does it have backoff + jitter?
   Does it respect context cancellation?

4. **Lifecycle:** Are background tasks tracked? Will they
   shut down gracefully? Can you detect leaks in tests?

5. **Telemetry:** Are events emitted or is logging the only
   observability? Can consumers attach their own handlers?

### Red Flags:

- `log.Info("something happened")` with no channel/audience
- Config read from environment at point-of-use (not validated)
- Retry logic duplicated in 5 places with different backoff
- Goroutines launched with `go func()` and no tracking
- No telemetry events — only log lines for observability

<!-- PATTERN_COMPLETE -->
