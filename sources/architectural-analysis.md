# Architectural Patterns from Top Repos

## CockroachDB: How to Organize 20,000 Files

### The 116-Package Principle

CockroachDB has 116 packages under `pkg/util/` averaging
**4 files each**. This is deliberate:

**Force:** A 2M-line codebase where developers work on
different subsystems simultaneously. If `pkg/util` were
5 big packages, every PR would conflict.

**Pattern:** One concept = one package. `circuit/` is 3
files (breaker, options, signal). `quotapool/` is 5 files.
`stop/` is 2 files. The package boundary IS the API
boundary — no internal debates about what is exported.

**Naming:** Single-concept nouns. No `helpers`, no
`common`, no `shared`. Every package name tells you what
it does: `cancelchecker`, `ctxgroup`, `syncutil`.

### Dependency Layering

```
sql → kv → storage → util
 ↓     ↓       ↓
 ↓     ↓    roachpb (protobuf types)
 ↓     ↓       ↓
 ↓     keys ← util
 ↓
 settings, config
```

**Critical insight:** `kv` imports from `sql` AND `sql`
imports from `kv`. They solved circular deps via
interfaces + callback registration — not by eliminating
the cycle. The `internal/` package provides the bridge.

`storage` imports `kv` (for transaction types) but `kv`
also imports `storage`. Again, interface boundaries break
the cycle at compile time.

**Lesson:** Perfect layering is impossible in distributed
databases. The real skill is knowing where to put the
interface that breaks the cycle.

### Error Handling at Scale

They use `github.com/cockroachdb/errors` — their own
library that extends stdlib `errors` with:

- **Error marks:** Tag errors with metadata without
  changing the error chain
- **Wrapping with causes:** `errors.Wrap(err, "context")`
- **Safe printing:** `redact.Sprint` for log-safe errors
- **Network encoding:** Errors serialize across RPC
  boundaries

**Pattern:** Errors are first-class data that flows through
the entire system, surviving serialization across nodes.
Not just strings — structured, typed, matchable.

### Circuit Breaker (not stdlib)

```go
type Breaker struct {
    mu struct {
        syncutil.RWMutex
        errAndCh *errAndCh  // stable Signal() results
        probing  bool
    }
}
```

**Key design:** `Signal()` returns a channel + error getter
(like `context.Done()` + `context.Err()`). The channel is
stable — closing it doesn't affect callers who already have
a reference. New callers get a new channel after reset.

**Force:** In a distributed DB, a broken replica should
fail-fast all pending requests, then probe for recovery.
Context cancellation isn't enough because you need to
distinguish "gave up waiting" from "system is broken."

### QuotaPool: Abstract Resource Allocation

```go
type Resource interface{}
type Request interface {
    Acquire(ctx context.Context, r Resource) (
        fulfilled bool, tryAgainAfter time.Duration)
    ShouldWait() bool
}
```

**Pattern:** The pool is generic over any resource type.
Concrete implementations include:
- `IntPool` — weighted semaphore with FIFO ordering
- Rate limiters (via `tryAgainAfter`)
- Token buckets

**Force:** Different subsystems need different quota types
but the same queueing/fairness semantics. Abstract once,
instantiate many.

---

## Prometheus: Interface-Driven Storage Architecture

### The Contract Layer

`storage/interface.go` defines **15+ interfaces** that
form the entire query/storage contract:

```
Storage (top level)
├── Appendable → Appender (write path)
├── Queryable → Querier (read path)
├── ChunkQueryable → ChunkQuerier (bulk read)
├── ExemplarStorage (exemplars)
└── Searcher (experimental)
```

**Force:** Prometheus must support:
- Local TSDB (the main implementation)
- Remote read/write (federation)
- Recording rules (virtual series)
- Testing (mock implementations)

All through the same interface. The contract layer is
the single point of truth for "what does storage mean."

### Compile-Time Interface Verification

```go
var _ storage.GetRef = &headAppender{}
var _ storage.Searcher = &blockBaseQuerier{}
```

Prometheus uses this pattern **8 times** in tsdb/ alone.
Every concrete type that claims to satisfy a storage
interface proves it at compile time.

**Why this matters at scale:** Storage interfaces evolve.
When `Searcher` was added, every type that should
implement it needed updating. The `var _` pattern makes
the compiler tell you what you missed.

### Plugin Discovery via Channel

```go
type Discoverer interface {
    Run(ctx context.Context, up chan<- []*targetgroup.Group)
}
```

**Brilliance:** The entire service discovery system is one
interface with one method. Consul, DNS, Kubernetes, AWS —
all implement `Run`. They push target groups through a
channel. The manager multiplexes.

**Force:** Prometheus supports 20+ discovery mechanisms.
Adding one should require zero changes to the core. The
channel-based push model means the manager never polls.

### Atomic File Operations

Block lifecycle uses filesystem conventions:
- `.tmp-for-creation` — incomplete write
- `.tmp-for-deletion` — incomplete delete

On startup, scan and clean up. No WAL needed for
block-level operations because rename is atomic on POSIX.

**Force:** TSDB blocks are large (hours of data). A WAL
for block operations would be overkill. The suffix
convention gives crash consistency with zero overhead.

---

## Ecto: Composability Through Data

### Query as Accumulating Struct

```elixir
defstruct prefix: nil, sources: nil, from: nil,
          joins: [], wheres: [], select: nil,
          order_bys: [], limit: nil, offset: nil,
          group_bys: [], updates: [], havings: [],
          preloads: [], distinct: nil, lock: nil,
          windows: [], with_ctes: nil
```

**Every query operation appends to a list or sets a
field.** Nothing is executed. The struct accumulates intent
until `Repo.all/Repo.one` triggers planning + execution.

**Force:** Queries must be composable (build in one
module, filter in another, paginate in a third). If
operations executed immediately, composition would require
the entire DB context at every step.

### Macro → Builder → Planner Pipeline

```
User writes: from(u in User, where: u.age > 18)
                     ↓
Macro expands: Builder.Filter.build(query, expr, env)
                     ↓
Builder produces: %Ecto.Query.BooleanExpr{...}
                     ↓
Planner resolves: types, bindings, params
                     ↓
Adapter generates: SQL string
```

Each builder module handles one clause type. There are
**15 builder modules** (from, join, filter, select, etc.).
The planner doesn't know about SQL — it resolves the
query struct into a normalized form that any adapter can
consume.

**Force:** Support multiple databases (Postgres, MySQL,
SQLite) with the same query language. The adapter is the
only part that knows SQL dialect.

### Protocol for Extensibility

`Ecto.Queryable` protocol lets you pass:
- A module atom (`User`) → resolved to schema query
- A string (`"users"`) → raw table
- A tuple (`{"filtered_users", User}`) → view + schema
- An `Ecto.Query` struct → identity

**Force:** `Repo.all(X)` should work with any "queryable
thing." New queryable types can be added without touching
Repo code.

---

## Oban: Architecture for Testability

### Engine Swap by Config

```elixir
def get_engine(%{engine: engine, testing: :disabled}), do: engine
def get_engine(%{testing: :inline}), do: Oban.Engines.Inline
def get_engine(%{testing: :manual}), do: engine
```

Three modes:
- **disabled** (production) — real engine
- **inline** (unit test) — execute in caller process
- **manual** (integration) — enqueue but don't execute

**Force:** Background jobs are inherently untestable
without process control. Rather than making tests async
(flaky), make the engine deterministic.

### Flat Supervision with Named Registry

```elixir
children = [
  {Notifier, conf: conf, name: Registry.via(name, Notifier)},
  {Nursery, conf: conf, name: Registry.via(name, Nursery)},
  {Peer, conf: conf, name: Registry.via(name, Peer)},
  {Sonar, conf: conf, name: Registry.via(name, Sonar)},
  {Harbor, conf: conf, name: Registry.via(name, Harbor)}
]
```

Every child gets its config via `conf:` and its identity
via `Registry.via`. This means:
- Multiple Oban instances can run in the same VM
- Tests can start isolated Oban supervisors
- No global state — everything is namespaced

**Force:** Libraries can't own global names. Enterprise
apps run multiple Oban instances (different repos,
different queues). The Registry pattern makes this
possible without process naming conflicts.

### Behaviour as Plugin Contract

```elixir
# Plugin must be a GenServer AND implement these:
@callback start_link([option()]) :: GenServer.on_start()
@callback validate([option()]) :: :ok | {:error, String.t()}
```

**Force:** Plugins need lifecycle management (start, stop,
crash recovery) AND configuration validation. By requiring
both a behaviour AND OTP compliance, Oban gets:
- Fault isolation (supervisor restarts crashed plugins)
- Config validation at startup (fail fast)
- No coupling (any GenServer works)

---

## Cross-Cutting Insights

### 1. Interfaces at Boundaries, Structs Internally

All four codebases define interfaces at system boundaries
(storage, engine, discovery) but use concrete types
internally. The interface is the published contract; the
struct is the implementation detail.

### 2. Config as Validated Struct, Not Map

Every system validates config at startup and stores it as
a typed struct. Never a raw map floating around.

### 3. Testing is an Architecture Decision

Oban's engine swap, CockroachDB's stopper tracking,
Prometheus's mock interfaces — testability isn't bolted on,
it's designed in from day one.

### 4. Composition via Data, Not Inheritance

Ecto queries accumulate as data. Prometheus discoverers
push through channels. CockroachDB quota requests are
data objects. Nobody uses class hierarchies.

### 5. The Cycle Problem is Solved with Interfaces

CockroachDB has circular dependencies between sql↔kv↔
storage. They break cycles with interface packages that
both sides depend on. This is the only way at scale.

### 6. Small Packages > Large Packages

CockroachDB: 4 files average per package.
Oban: focused modules (engine, worker, plugin).
Ecto: one builder per clause type.
The package boundary forces you to define the API.

<!-- PATTERN_COMPLETE -->
