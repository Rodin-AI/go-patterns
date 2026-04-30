# Patterns Extracted from cockroachdb/cockroach

## Pattern: Stopper for Goroutine Lifecycle

**Source:** `pkg/util/stop/stopper.go`
**Category:** concurrency

**What:** A dedicated struct that manages the lifecycle of
all goroutines in a component: tracks active tasks, refuses
new work during shutdown (quiesce), waits for completion,
then runs closers.

**Why:** In distributed systems, clean shutdown is critical.
You need to: (1) stop accepting new work, (2) finish
in-flight work, (3) release resources in order. The Stopper
centralizes this instead of scattering shutdown logic across
every goroutine.

**Example:**

```go
type Stopper struct {
    quiescer chan struct{} // closed when quiescing
    stopped  chan struct{} // closed when fully stopped
    mu struct {
        syncutil.RWMutex
        _numTasks int32
        quiescing, stopping bool
        closers []Closer
    }
}

// RunAsyncTask refuses new work during quiesce
func (s *Stopper) RunAsyncTask(ctx context.Context,
    taskName string, f func(context.Context)) error {
    if !s.addTask() {
        return ErrUnavailable
    }
    go func() {
        defer s.decTask()
        f(ctx)
    }()
    return nil
}
```

**When to use:** Any server or subsystem that spawns
goroutines and needs graceful shutdown. Especially in
long-running services where leaked goroutines cause
resource exhaustion.

**When NOT to use:** Simple programs with a single main
goroutine. Or when `errgroup` with context cancellation
suffices for the shutdown coordination.

---

## Pattern: Tracked Lifecycle with Leak Detection

**Source:** `pkg/util/stop/stopper.go`
**Category:** testing

**What:** Register every Stopper instance in a global
tracker. In tests, call `PrintLeakedStoppers(t)` to detect
any Stopper that was created but never stopped — indicating
a resource leak.

**Why:** Distributed systems have complex lifecycle graphs.
A forgot-to-stop bug silently leaks goroutines and
connections. The tracker makes leaks fail-loud in tests
without requiring careful manual cleanup.

**Example:**

```go
var trackedStoppers struct {
    syncutil.Mutex
    stoppers []stopperWithStack
}

func register(s *Stopper) {
    trackedStoppers.Lock()
    trackedStoppers.stoppers = append(...)
    trackedStoppers.Unlock()
}

func PrintLeakedStoppers(t testing.TB) {
    for _, tracked := range trackedStoppers.stoppers {
        t.Errorf("leaked stopper, created at:\n%s",
            tracked.createdAt)
    }
}
```

**When to use:** Any resource that must be explicitly
closed/stopped and where forgetting to do so causes silent
degradation.

**When NOT to use:** Resources with finalizers or GC-safe
cleanup. Adds global state — only for testing.

---

## Pattern: Quiesce Then Stop (Two-Phase Shutdown)

**Source:** `pkg/util/stop/stopper.go`
**Category:** concurrency

**What:** Shutdown has two explicit phases: (1) Quiesce —
refuse new work, wait for in-flight to finish; (2) Stop —
run closers, signal done. Components observe
`ShouldQuiesce` channel alongside context.

**Why:** One-phase shutdown (just cancel context) loses
in-flight work. Two-phase gives running tasks time to
complete while preventing new work from starting. The
explicit channel (vs just context) lets components
distinguish "winding down" from "dead."

**Example:**

```go
func worker(s *Stopper, ctx context.Context) {
    for {
        select {
        case <-s.ShouldQuiesce():
            return // graceful: finish current, exit
        case <-ctx.Done():
            return // hard cancel
        case work := <-workChan:
            process(work)
        }
    }
}
```

**When to use:** Servers handling requests where you want
zero-downtime deploys (drain then stop). Load balancers,
RPC servers, queue consumers.

**When NOT to use:** Batch jobs or CLIs where immediate
exit is fine.

---

## Pattern: CloserFn Adapter

**Source:** `pkg/util/stop/stopper.go`
**Category:** concurrency

**What:** Define a `Closer` interface with one method
(`Close()`), plus a `CloserFn` type that adapts any
function into a Closer.

**Why:** The adapter pattern (like `http.HandlerFunc`)
avoids forcing users to define a struct just to implement
a one-method interface. Cleanup functions can be registered
directly.

**Example:**

```go
type Closer interface { Close() }
type CloserFn func()
func (f CloserFn) Close() { f() }

// Usage:
stopper.AddCloser(stop.CloserFn(func() {
    conn.Close()
}))
```

**When to use:** Any one-method interface where callers
often have a simple function they want to register.

**When NOT to use:** Interfaces with >1 method, or when
the implementation needs state beyond a closure.

<!-- PATTERN_COMPLETE -->
