# Stdlib vs Kubernetes: Where K8s Does Things Differently

## 1. Concurrency: Channels vs. Condition Variables + Sets

### Stdlib approach
Go stdlib idiom: communicate via channels. A worker pool typically uses a `chan T` for work items.

```go
// Typical stdlib pattern:
jobs := make(chan Job, 100)
for i := 0; i < workers; i++ {
    go func() {
        for job := range jobs {
            process(job)
        }
    }()
}
```

### Kubernetes approach
The workqueue uses `sync.Cond` + `sets.Set` instead of channels.

**Source:** `staging/src/k8s.io/client-go/util/workqueue/queue.go` (lines 190–300)

```go
type Typed[t comparable] struct {
    queue      Queue[t]
    dirty      sets.Set[t]     // Items needing processing
    processing sets.Set[t]     // Items currently being processed
    cond       *sync.Cond      // Notification mechanism
}
```

### Why K8s is different
Channels don't deduplicate. If you send the same key twice to a channel, it gets processed twice. K8s needs:
- **Deduplication**: same object modified 5 times → process once with latest state
- **Re-entrant marking**: if modified during processing, re-queue after Done()
- **Inspection**: can query queue length, processing count for metrics

A channel-based design would require a separate dedup layer anyway, losing its simplicity advantage.

---

## 2. Error Handling: Single Errors vs. Error Aggregation

### Stdlib approach
Return a single error. Wrap with `fmt.Errorf("...: %w", err)`.

### Kubernetes approach
**Source:** `staging/src/k8s.io/apimachinery/pkg/util/errors/errors.go` (lines 34–100)

```go
type Aggregate interface {
    error
    Errors() []error
    Is(error) bool
}

func NewAggregate(errlist []error) Aggregate {
    // Filters nils, deduplicates messages
}
```

### Why K8s is different
A controller sync often does multiple operations (create 3 pods, update 2 services). You want to attempt all of them, not fail fast on the first error. Error aggregation collects all failures so the user sees the full picture.

**Also**: the `Aggregate` interface properly implements `errors.Is()` by checking if *any* contained error matches — which `errors.Join` didn't originally support well.

---

## 3. Retry: stdlib has nothing; K8s has structured retry

### Stdlib approach
There's no retry utility in stdlib. You write your own loop.

### Kubernetes approach
**Source:** `staging/src/k8s.io/client-go/util/retry/util.go` (lines 30–100)

```go
var DefaultRetry = wait.Backoff{
    Steps:    5,
    Duration: 10 * time.Millisecond,
    Factor:   1.0,
    Jitter:   0.1,
}

func RetryOnConflict(backoff wait.Backoff, fn func() error) error {
    // Retries only on HTTP 409 Conflict
    return OnError(backoff, errors.IsConflict, fn)
}
```

### Why K8s is different
In a distributed system, retries are *the* primary reliability mechanism. Stdlib doesn't provide them because stdlib targets single-machine programs. Kubernetes needs:
- Configurable backoff (steps, factor, jitter, cap)
- Condition-based retry (retry only on specific error types)
- Context-aware cancellation

---

## 4. Polling: time.Ticker vs. Contextual Loop With Crash Protection

### Stdlib approach
```go
ticker := time.NewTicker(interval)
defer ticker.Stop()
for range ticker.C {
    doWork()
}
```

### Kubernetes approach
**Source:** `staging/src/k8s.io/apimachinery/pkg/util/wait/backoff.go` (lines 240–260), `loop.go` (lines 38–80)

```go
func BackoffUntilWithContext(ctx context.Context, f func(ctx context.Context), backoff BackoffManager, sliding bool) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
        }
        if !sliding { t = backoff.Backoff() }
        func() {
            defer runtime.HandleCrashWithContext(ctx)
            f(ctx)
        }()
        if sliding { t = backoff.Backoff() }
        // ... wait for timer or context cancellation
    }
}
```

### Why K8s is different
- **Crash protection**: a panic in `doWork()` shouldn't kill the whole process
- **Sliding vs non-sliding**: controls whether interval includes execution time
- **Context cancellation**: allows clean shutdown
- **Jitter**: prevents thundering herd when many controllers sync at similar intervals
- **Double-check for cancellation**: Go's select is non-deterministic, so short timers can "win" against a cancelled context

---

## 5. Graceful Shutdown: http.Server.Shutdown vs. Multi-Phase Orchestration

### Stdlib approach (net/http)
**Source:** `/tmp/go-src/src/net/http/server.go` (lines 3221–3260)

```go
func (s *Server) Shutdown(ctx context.Context) error {
    s.inShutdown.Store(true)
    s.closeListenersLocked()
    // Poll for idle connections
    for {
        if s.closeIdleConns() { return nil }
        select {
        case <-ctx.Done(): return ctx.Err()
        case <-timer.C: timer.Reset(nextPollInterval())
        }
    }
}
```

### Kubernetes approach
**Source:** `pkg/controller/deployment/deployment_controller.go` (lines 171–196)

```go
func (dc *DeploymentController) Run(ctx context.Context, workers int) {
    defer utilruntime.HandleCrash()
    dc.eventBroadcaster.StartStructuredLogging(3)
    dc.eventBroadcaster.StartRecordingToSink(...)
    defer dc.eventBroadcaster.Shutdown()

    var wg sync.WaitGroup
    defer func() {
        dc.queue.ShutDown()  // Stop accepting new work
        wg.Wait()            // Wait for workers to finish
    }()

    // Gate: don't start until caches are synced
    if !cache.WaitForNamedCacheSyncWithContext(ctx, ...) { return }

    for i := 0; i < workers; i++ {
        wg.Go(func() {
            wait.UntilWithContext(ctx, dc.worker, time.Second)
        })
    }
    <-ctx.Done()  // Block until context cancelled
}
```

### Why K8s is different
Stdlib's shutdown is **reactive** (wait for connections to drain). Kubernetes' shutdown is **multi-phase orchestrated**:
1. Stop accepting new events (close watch connections)
2. Drain the work queue (process remaining items)
3. Wait for in-flight syncs to complete
4. Shut down event recorders

The queue's `ShutDownWithDrain()` is the K8s-specific innovation: it waits until all in-flight items call `Done()`.

---

## 6. Type Systems: interfaces vs. Runtime Type Registry

### Stdlib approach
Interfaces for polymorphism. If you need to serialize, use `encoding/json` with struct tags.

### Kubernetes approach
A full runtime type registry (Scheme) that maps between GVK strings and Go types.

**Source:** `staging/src/k8s.io/apimachinery/pkg/runtime/scheme.go`

### Why K8s is different
Stdlib's `encoding/json` requires knowing the concrete type at compile time. Kubernetes must:
- Deserialize objects from the network without knowing their type in advance
- Convert between API versions (`v1beta1.Deployment` → `v1.Deployment`)
- Support third-party types (CRDs) that don't exist at compile time
- Apply defaulting and validation based on type metadata

This forces a **runtime type system layered on top of Go's static types**.

---

## 7. Testing: httptest vs. Fake Clients + Reactors

### Stdlib approach
`net/http/httptest` provides a test server. You make real HTTP calls against it.

### Kubernetes approach
Fake clientsets with reactor chains:
```go
// Generated fake clients intercept API calls
fakeClient := fake.NewSimpleClientset(existingObjects...)
fakeClient.PrependReactor("create", "pods", func(action testing.Action) (bool, runtime.Object, error) {
    // Custom test behavior
    return true, nil, fmt.Errorf("simulated error")
})
```

### Why K8s is different
Testing a controller doesn't require a running API server. The fake client + informer pattern lets you:
- Inject specific starting states
- Simulate failures at specific operations
- Run synchronously (no network delay)
- Test the controller logic in isolation

---

## 8. Lifecycle: main() returns vs. Infinite Reconciliation

### Stdlib pattern
Programs start, do work, return.

### Kubernetes pattern
Controllers start and run *forever*, continuously reconciling.

```go
// The fundamental difference: stdlib programs terminate, controllers don't
func main() {
    // Stdlib: do work, exit
    result := compute()
    fmt.Println(result)
}

// Kubernetes: infinite loop with eventual consistency
func (c *Controller) Run(ctx context.Context) {
    // Run forever until context cancelled
    for i := 0; i < workers; i++ {
        go wait.UntilWithContext(ctx, c.worker, time.Second)
    }
    <-ctx.Done()
}
```

### Why K8s is different
The real world is adversarial. Networks fail, nodes die, humans make mistakes. A one-shot program can't handle drift. The reconciliation loop is Kubernetes' answer to the CAP theorem: you can't guarantee consistency in a single call, but you can achieve it *eventually* through repetition.

---

## 9. Shared State: Package-Level vs. Shared Informer Cache

### Stdlib approach
Package-level variables, or pass state through function parameters.

### Kubernetes approach
The SharedInformerFactory creates a single in-memory cache per resource type, shared by all controllers in the process.

```go
// All controllers share ONE watch and ONE cache per resource:
informerFactory := informers.NewSharedInformerFactory(client, resyncPeriod)
deployInformer := informerFactory.Apps().V1().Deployments()

// Controller A and B both get events from the same informer
deployInformer.Informer().AddEventHandler(controllerA)
deployInformer.Informer().AddEventHandler(controllerB)
```

### Why K8s is different
Without sharing:
- 20 controllers × watch for Pods = 20 TCP connections to API server
- 20 copies of all Pods in memory

With SharedInformerFactory:
- 1 TCP connection for Pods
- 1 copy in memory
- Events fanned out to all registered handlers

---

## 10. Configuration: Flags/Env vs. Feature Gates

### Stdlib approach
`flag` package, environment variables, config files.

### Kubernetes approach
Feature gates: a versioned, lifecycle-aware configuration system.

### Why K8s is different
Stdlib's flag package is for a single binary. Kubernetes has:
- Hundreds of features in various stages of maturity
- Features that must be consistent across control plane components
- Features that need to be enabled/disabled without redeployment
- Features with dependencies on other features
- Automated testing that exercises all combinations

Feature gates encode *maturity* (alpha/beta/GA) alongside the boolean value, something `flag.Bool` can never express.
