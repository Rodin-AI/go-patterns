# Patterns from Temporal (temporalio/temporal)

Source: github.com/temporalio/temporal
Analyzed: 2026-04-30
Commits: 8,958 | Contributors: 290

---

## 1. Effect Buffer (Transactional Side Effects)

**Location:** `common/effect/buffer.go`

Buffer side effects during a transaction, apply after
commit, rollback (in reverse order) after failure.

```go
type Buffer struct {
    effects []func(context.Context)
    cancels []func(context.Context)
}

func (b *Buffer) OnAfterCommit(effect func(ctx)) {
    b.effects = append(b.effects, effect)
}

func (b *Buffer) OnAfterRollback(effect func(ctx)) {
    b.cancels = append(b.cancels, effect)
}

func (b *Buffer) Apply(ctx context.Context) bool {
    b.cancels = nil
    for _, effect := range b.effects {
        effect(ctx) // FIFO order
    }
    b.effects = nil
    return true
}

func (b *Buffer) Cancel(ctx context.Context) bool {
    b.effects = nil
    for i := len(b.cancels) - 1; i >= 0; i-- {
        b.cancels[i](ctx) // LIFO (reverse) order
    }
    b.cancels = nil
    return true
}
```

**When to use:** Any operation that has side effects
(notifications, cache invalidation, external calls)
that should only execute if the primary transaction
succeeds.

**When NOT to use:** Simple operations where failure
leaves no cleanup needed. Over-engineering for
functions with a single side effect.

---

## 2. Soft Assertions (Log, Don't Crash)

**Location:** `common/softassert/softassert.go`

Assert invariants in production code without panicking.
Log the violation so tests catch it but production
continues serving.

```go
func That(logger log.Logger, condition bool,
    staticMessage string, tags ...tag.Tag) bool {
    if !condition {
        logger.Error("failed assertion: "+staticMessage,
            append([]tag.Tag{tag.FailedAssertion},
                tags...)...)
    }
    return condition
}
```

Usage at call sites:
```go
if !softassert.That(logger, obj.State == "ready",
    "object not ready before dispatch") {
    return // or take recovery action
}
```

**The philosophy (from PR #7411):** "Why not panic?
Maybe in the future. For now, we're happy with finding
these failed assertions in functional tests."

**When to use:** Invariants that should always hold
but where crashing production is worse than logging.
Distributed systems where one node's invariant
violation shouldn't cascade.

**When NOT to use:** Safety-critical paths where
corruption would be worse than downtime. Better to
crash than serve corrupt data.

---

## 3. Type-Safe State Transitions

**Location:** `service/history/hsm/sm.go`

State machines with compile-time source state
validation and typed events.

```go
type Transition[S comparable, SM StateMachine[S], E any] struct {
    Sources     []S
    Destination S
    apply       func(SM, E) (TransitionOutput, error)
}

func (t Transition[S, SM, E]) Apply(sm SM, event E) (TransitionOutput, error) {
    if !slices.Contains(t.Sources, sm.State()) {
        return TransitionOutput{},
            fmt.Errorf("%w from %v: %v",
                ErrInvalidTransition, sm.State(), event)
    }
    sm.SetState(t.Destination)
    return t.apply(sm, event)
}
```

**When to use:** Any system with defined state
lifecycles — workflow engines, order processing,
connection state, protocol implementations.

**When NOT to use:** Simple boolean flags, systems
where state changes are free-form or user-defined.

---

## 4. Mutable vs Immutable Context (Type-Level Access Control)

**Location:** `chasm/context.go`

Separate `Context` (read-only) from `MutableContext`
(read-write) at the type level. Functions that only
read take `Context`; functions that mutate take
`MutableContext`.

```go
// Read-only operations
func (h *Handler) Check(ctx chasm.Context, c Component) (bool, error)

// Mutation operations
func (h *Handler) Execute(ctx chasm.MutableContext, c Component) error
```

**From PR discussion (Sushisource):** "I think I prefer
them separate, because what happens if you mutate
something and then say 'not ready'? That would be some
weird violation that shouldn't be possible, and separate
contexts enforces that at the type level."

**When to use:** Any system where read vs write access
matters — databases, caches, state machines, APIs with
both query and mutation.

**When NOT to use:** Simple CRUD where every operation
is both a read and a write.

---

## 5. Goroutine Handle (Safe Lifecycle)

**Location:** `common/goro/goro.go`

A handle to a goroutine that supports safe multi-stop,
context cancellation, and error collection.

```go
h := goro.NewHandle(ctx)
h.Go(func(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case task := <-ch:
            process(task)
        }
    }
})

// Safe to call multiple times:
h.Cancel()
err := h.Wait()
```

**Born from:** PR #1892, fixing a double-close panic in
the task writer. The old code used a raw `chan struct{}`
for shutdown, which panics if closed twice.

**When to use:** Any long-lived goroutine that needs
clean shutdown, error reporting, or may be stopped from
multiple places.

**When NOT to use:** Fire-and-forget goroutines, or
goroutines managed by higher-level frameworks
(errgroup, worker pools).

---

## 6. Dynamic Config with Type-Safe Generics

**Location:** `common/dynamicconfig/`

566 settings, each declared as a typed constant with
default value, description, and precedence resolution.

```go
var WorkflowMaxTimeout = NewNamespaceDurationSetting(
    "limit.maxWorkflowTimeout",
    24*365*time.Hour,
    `Maximum timeout for a workflow execution.`,
)

// Consumer side:
timeout := dc.GetDurationProperty(
    dynamicconfig.WorkflowMaxTimeout,
    namespace,
)()
```

Resolution order: task queue → namespace → global.
Uses `weak.Pointer` cache for GC-friendly memoization.

**When to use:** Large systems with many operational
knobs that need to change without restart. Multi-tenant
systems where different tenants need different limits.

**When NOT to use:** Simple services with <10 config
values. Static configuration is simpler and more
predictable.

---

## 7. Composable Predicates (Filter Algebra)

**Location:** `common/predicates/`

Generic predicate interface with And/Or/Not composition
and automatic flattening.

```go
type Predicate[T any] interface {
    Test(T) bool
    Equals(Predicate[T]) bool
    Size() int
}

// Usage:
filter := predicates.And(
    predicates.Not(isExpired),
    predicates.Or(isHighPriority, isRetryable),
)
```

Flattening: `And(And(a, b), c)` → `And(a, b, c)`.
Short-circuit: `And(Empty, anything)` → `Empty`.

**When to use:** Task queues, query builders, access
control rules — anywhere filters compose.

**When NOT to use:** Single predicate checks, simple
if-statements.

---

## 8. Persistence Plugin Registration (init pattern)

**Location:** `common/persistence/sql/`

```go
func init() {
    sql.RegisterPlugin("postgres12", &plugin{
        driver: &driver.PQDriver{},
    })
}
```

Import-driven registration. The main binary imports the
plugins it wants; each plugin's init() registers it.

**When to use:** Database drivers, encoding formats,
protocol implementations — anything with a fixed set
of implementations selected at compile time.

**When NOT to use:** When you need runtime plugin
discovery (use a factory pattern instead). When the
number of implementations is small and stable (just
use a switch).

---

## 9. ShutdownOnce (CAS-Based Safe Close)

**Location:** `common/channel/shutdown_once.go`

```go
func (c *ShutdownOnceImpl) Shutdown() {
    if atomic.CompareAndSwapInt32(
        &c.status, statusOpen, statusClosed) {
        close(c.channel)
    }
}
```

Wraps the "close of closed channel" panic with a CAS
guard. Safe to call from multiple goroutines.

**When to use:** Any shared shutdown signal where
multiple goroutines might initiate shutdown. Replaces
`sync.Once` when you also need a channel for select.

**When NOT to use:** When only one goroutine ever
triggers shutdown. When `context.WithCancel` suffices.

<!-- PATTERN_COMPLETE -->
