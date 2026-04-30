# Ecosystem-Level Patterns: How Codebases Present to Consumers

## The Three Questions

For each codebase, ask:
1. How do consumers **extend** it? (What interfaces/behaviours
   do they implement?)
2. How do consumers **compose** with it? (What does day-to-day
   usage look like?)
3. What does it deliberately **NOT do**? (What forces shaped
   those refusals?)

---

## CockroachDB: Errors as First-Class Distributed Data

### Extension Points

CockroachDB is not a library — it is a system. Consumers
extend it through:
- **SQL builtins** (function registration)
- **Storage engines** (via pebble interface)
- **Service discovery** (not user-extensible — closed)

The interesting pattern is how errors flow from storage
through KV through SQL to the client.

### Error Architecture (ecosystem-level idiom)

```
Storage error → encoded via cockroachdb/errors →
  KV wraps with context → serialized across gRPC →
  SQL decodes → maps to pgcode → wire protocol to client
```

**Key design decisions:**

1. **Errors have priority.** `ErrPriority()` ranks errors so
   the system knows which to surface when multiple things
   fail simultaneously. Transaction abort > restart >
   unambiguous error > non-retriable.

2. **Errors survive serialization.** `EncodeError` /
   `DecodeError` serialize errors across RPC boundaries.
   The error that originated on node 3 arrives at node 1
   with its full cause chain intact.

3. **Errors map to pg codes.** Every internal error maps to
   a Postgres error code that clients understand. This is
   the *ecosystem contract* — clients write
   `if pgcode == '40001' { retry }`.

**What this teaches:** In a distributed system, an error
isn't a string — it's a data object with identity,
priority, serializability, and a consumer-facing code.
Design your error types for the *consumer*, not the
*producer*.

### Deliberate Absences

- **No dependency injection framework.** Config structs
  passed explicitly. 1178-line `StoreConfig` struct, but
  it's all data — no framework magic.
- **No context.Background() on hot paths.** 144 uses in
  kvserver, but auditable — each justified in comments.
- **No functional options.** CockroachDB uses config
  structs universally. The Option interface in stopper is
  the exception, not the rule.

### Test Architecture

- **TestMain in every package.** Sets up security certs,
  random seeds, and test server factories.
- **Goroutine leak detection.** `leaktest.AfterTest(t)()`
  at the start of every test. Detects leaked goroutines
  by diffing goroutine stacks before/after.
- **Stopper leak detection.** Every Stopper is tracked
  globally; `PrintLeakedStoppers(t)` in TestMain catches
  forgot-to-stop bugs.
- **`//go:generate` for test setup.** Codegen tool
  (`add-leaktest.sh`) auto-adds leak checks to every
  test file.

**What this teaches:** At scale, the most important test
infrastructure isn't assertions — it's resource leak
detection. Every goroutine, every connection, every
Stopper is tracked and verified to be cleaned up.

---

## Prometheus: The One-Method Interface Contract

### Extension Points

Prometheus is extended through:
- **Service discovery** (30 implementations, 1 interface)
- **Storage** (remote read/write adapters)
- **Exporters** (client_golang metrics)

### The Discoverer Pattern (ecosystem-level idiom)

```go
type Discoverer interface {
    Run(ctx context.Context, up chan<- []*targetgroup.Group)
}
```

This is **one method**. Thirty implementations. The
channel-based push model means:
- The discoverer controls timing (not polled)
- The manager multiplexes without knowing implementations
- Adding a new discovery source = implement Run, register

**Registration via init():**
```go
func init() {
    discovery.RegisterConfig(&SDConfig{})
}
```

This is the classic Go plugin pattern. Import the package
→ init registers it → the system discovers it at startup.

**What this teaches:** The smallest possible interface
creates the largest possible ecosystem. One method + one
channel = 30 implementations without coordination.

### Storage Contract (15 interfaces, 1 file)

All of Prometheus's storage contract lives in
`storage/interface.go`. This is the:
- Read path: `Queryable → Querier → SeriesSet → Series`
- Write path: `Appendable → Appender`
- Extension: `ExemplarAppender`, `MetadataUpdater`

**Key:** Every implementation proves satisfaction at
compile time with `var _ storage.Searcher = &type{}`.
When the contract evolves, the compiler finds every
broken implementation.

### Deliberate Absences

- **No generics in storage interfaces.** Despite Go 1.20+
  support. The interfaces predate generics and adding them
  would break all existing implementations.
- **No dependency injection.** Direct struct construction
  everywhere. Testability through interface satisfaction,
  not framework wiring.
- **Almost no functional options.** Only in leaf packages
  (chunk writer, parser). Core APIs use config structs.
- **No goroutine leak in production code.** `goleak` in
  tests, `TolerantVerifyLeak` with explicit allowlist for
  known third-party leaks.

### Test Architecture

- **`TolerantVerifyLeak`** — goroutine leak detection with
  allowlist for known third-party leaks (opencensus, klog)
- **Mock implementations of every interface** — defined
  right in `storage/interface.go` next to the real ones
- **Golden file tests** in PromQL evaluation

---

## Ecto: Composability as Architectural Principle

### Extension Points

Consumers extend Ecto through:
- **Custom types** (7 callbacks: cast, load, dump, equal?,
  embed_as, autogenerate, type)
- **Adapters** (Queryable, Schema, Transaction, Storage —
  4 behaviour modules)
- **Protocols** (`Ecto.Queryable` — anything can become a
  query)

### The NotLoaded Sentinel (ecosystem-level idiom)

```elixir
defmodule Ecto.Association.NotLoaded do
  defstruct [:__field__, :__owner__, :__cardinality__]
end
```

Ecto **refuses to lazy-load associations**. If you access
`user.posts` without preloading, you get a `NotLoaded`
struct — not nil, not an empty list, not a database query.

**Why this is an ecosystem decision:**
- Forces consumers to be explicit about data needs
- Prevents N+1 queries by making them impossible
- Makes the data boundary visible in code

This is a *consumer-hostile* decision that makes
*systems built on Ecto* dramatically better. The library
optimizes for the 1000th user, not the first-day
experience.

### Query Composition (ecosystem-level idiom)

Every query clause appends to a list in the Query struct.
Nothing executes. The Query is pure data that accumulates
intent.

**Consumer impact:** You can build queries across module
boundaries:

```elixir
# Module A builds the base
def active_users, do: from(u in User, where: u.active)

# Module B adds pagination
def paginate(query, page, size) do
  query
  |> limit(^size)
  |> offset(^((page - 1) * size))
end

# Module C adds authorization
def visible_to(query, role) do
  where(query, [u], u.role in ^roles_for(role))
end
```

Each module is independent. They compose because queries
are data, not effects.

### Adapter Architecture

```
Ecto.Repo.all(query)
  → Planner resolves types, bindings
  → Adapter.prepare/2 produces {cache, prepared}
  → Adapter.execute/5 runs against DB
  → Adapter.loaders/2 converts back to Elixir types
```

The adapter is the ONLY part that knows SQL. Ecto core
is database-agnostic. This is why the same code works on
Postgres, MySQL, SQLite, and custom stores.

### Deliberate Absences

- **No lazy loading.** `NotLoaded` struct instead.
- **No global state.** Per-repo config, per-repo process.
- **No query caching at library level.** The adapter
  caches prepared statements; Ecto doesn't.
- **No connection to schema naming.** `schema "legacy_tbl"`
  is independent of `defmodule NewUser`.

---

## Oban: Designing for Testability First

### Extension Points

Consumers extend Oban through:
- **Workers** (`perform/1` — the job logic)
- **Plugins** (GenServer + validate callback)
- **Engines** (entire backend swap)
- **Notifiers** (pub/sub mechanism)
- **Peers** (leader election)

### The Worker Result Type (ecosystem-level idiom)

```elixir
@type result ::
  :ok
  | {:ok, ignored :: term()}
  | {:error, reason :: term()}
  | {:cancel, reason :: term()}
  | {:snooze, period :: Period.t()}
```

Five possible outcomes, each with distinct semantics:
- `:ok` → success, remove from queue
- `{:error, reason}` → retry (respects max_attempts)
- `{:cancel, reason}` → permanent failure, don't retry
- `{:snooze, period}` → reschedule for later

**Ecosystem impact:** Every worker author makes an
explicit decision about failure semantics. "What should
happen when this fails?" is answered in the type system,
not in configuration.

### Contextual Backoff (ecosystem-level idiom)

```elixir
def backoff(%Job{attempt: attempt, unsaved_error: err}) do
  case err.reason do
    %RateLimitError{retry_after: ms} -> ms
    _ -> trunc(:math.pow(attempt, 4) + jitter())
  end
end
```

The error that caused the failure is available to the
backoff calculation. Different errors → different retry
strategies. This is impossible in systems where backoff
is configured globally.

### Testing Design (ecosystem-level idiom)

Three testing modes via config:
- **`:inline`** — execute jobs synchronously in tests
- **`:manual`** — enqueue but don't execute
- **`:disabled`** — production behavior

Plus `use Oban.Testing` which provides:
- `assert_enqueued/1` — verify job was queued
- `refute_enqueued/1` — verify job was NOT queued
- `perform_job/2` — execute a job manually in tests
- `all_enqueued/1` — list all matching jobs

**Ecosystem impact:** Every Oban consumer gets
deterministic, fast, isolated tests for free. No sleep,
no polling, no flaky async assertions.

### Deliberate Absences

- **No global process names.** Registry.via everywhere —
  multiple Oban instances can coexist.
- **No direct DB coupling in workers.** Workers receive a
  Job struct; they don't import Repo.
- **No implicit retries.** max_attempts is explicit per
  worker. No "retry forever" default.
- **No built-in rate limiting in OSS.** That is a Pro
  feature — deliberate business boundary.

---

## Cross-Cutting: What "Idiomatic" Means at Ecosystem Level

### 1. The Consumer Contract is the API

Not the functions you export — the *experience* of
building on your system:
- CockroachDB: "Your errors will be pg-codes, always"
- Prometheus: "Implement Run(), get discovery for free"
- Ecto: "Queries are data; loading is always explicit"
- Oban: "Return a result type; testing is built in"

### 2. Deliberate Absences Define Character

What a system refuses to do is as important as what it
does:
- Ecto refuses lazy loading → forces explicit data needs
- Oban refuses global names → enables multi-instance
- Prometheus refuses DI frameworks → keeps simplicity
- CockroachDB refuses context.Background on hot paths →
  forces timeout discipline

### 3. Testability is Never Retrofitted

Every system that tests well designed testing in from the
start:
- CockroachDB: leak detection, stopper tracking
- Prometheus: goroutine leak verification, mock interfaces
- Ecto: adapter abstraction, embedded schemas for testing
- Oban: engine swap, testing modes, assertion helpers

### 4. Extension Points Define the Ecosystem Size

- Prometheus: 1 interface, 30 discoverers
- Ecto: 7 type callbacks, hundreds of custom types
- Oban: Worker behaviour + 5 engine callbacks

**Smaller interface → larger ecosystem.** The less you
demand from implementors, the more you get.

<!-- PATTERN_COMPLETE -->
