# Kubernetes-Specific Patterns

## 1. Controller / Reconciler Pattern

**Source:** `pkg/controller/deployment/deployment_controller.go` (lines 65–530)

### What it does
The controller pattern is the central design pattern of Kubernetes. Every controller watches a set of resources, maintains a work queue, and reconciles desired state with actual state through a sync loop.

### Why
Distributed systems can't guarantee that a single API call will bring the world to desired state. The controller pattern provides eventual consistency by continuously reconciling — it handles missed events, partial failures, and concurrent modifications.

### Structure

```go
// pkg/controller/deployment/deployment_controller.go:65-95
type DeploymentController struct {
    rsControl controller.RSControlInterface
    client    clientset.Interface

    // Testability: sync handler is injectable
    syncHandler func(ctx context.Context, dKey string) error
    enqueueDeployment func(deployment *apps.Deployment)

    // Listers: read from local cache, not API server
    dLister  appslisters.DeploymentLister
    rsLister appslisters.ReplicaSetLister
    podLister corelisters.PodLister

    // Synced funcs: gate processing until caches are warm
    dListerSynced  cache.InformerSynced
    rsListerSynced cache.InformerSynced

    // Work queue: rate-limited, deduplicating
    queue workqueue.TypedRateLimitingInterface[string]
}
```

### The Canonical Worker Loop

```go
// pkg/controller/deployment/deployment_controller.go:481-515
func (dc *DeploymentController) worker(ctx context.Context) {
    for dc.processNextWorkItem(ctx) {
    }
}

func (dc *DeploymentController) processNextWorkItem(ctx context.Context) bool {
    key, quit := dc.queue.Get()
    if quit {
        return false
    }
    defer dc.queue.Done(key)

    err := dc.syncHandler(ctx, key)
    dc.handleErr(ctx, err, key)
    return true
}

func (dc *DeploymentController) handleErr(ctx context.Context, err error, key string) {
    if err == nil || errors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
        dc.queue.Forget(key)  // Success: clear rate limiter
        return
    }
    if dc.queue.NumRequeues(key) < maxRetries {
        dc.queue.AddRateLimited(key)  // Retry with backoff
        return
    }
    utilruntime.HandleError(err)
    dc.queue.Forget(key)  // Give up after maxRetries
}
```

### Key Properties
1. **Level-triggered, not edge-triggered** — the sync loop reads current state, not diffs
2. **Idempotent** — running sync twice produces the same result
3. **Key-based deduplication** — the workqueue coalesces multiple events for the same object
4. **Bounded retries** — exponential backoff with a max retry count (15 retries = ~82s max delay)

---

## 2. Informer + Cache + Workqueue Combo

**Source:** `staging/src/k8s.io/client-go/tools/cache/shared_informer.go` (lines 144–283), `staging/src/k8s.io/client-go/informers/factory.go` (lines 57–250)

### What it does
The Informer provides a local read cache of API server state, backed by a List+Watch connection. The SharedInformerFactory ensures only one informer per resource type exists per process, preventing duplicate watches.

### Why
- **Reduces API server load**: controllers read from local cache (Listers) instead of hitting the API
- **Reduces latency**: events are delivered via callbacks, no polling
- **Memory efficiency**: shared informers prevent N controllers from opening N watches

### Architecture

```
API Server
    │
    ├── List (initial sync)
    │
    └── Watch (streaming updates)
            │
            ▼
    SharedIndexInformer
        ├── local Store (thread-safe cache)
        ├── Indexer (secondary indexes)
        └── Event Handlers → [Controller1, Controller2, ...]
                                    │
                                    ▼
                              WorkQueue
                                    │
                                    ▼
                              worker goroutines
```

### SharedInformerFactory Pattern

```go
// staging/src/k8s.io/client-go/informers/factory.go:57-77
type sharedInformerFactory struct {
    client           kubernetes.Interface
    lock             sync.Mutex
    informers        map[reflect.Type]cache.SharedIndexInformer
    startedInformers map[reflect.Type]bool
    wg               sync.WaitGroup
    shuttingDown     bool
}
```

### Registration via Event Handlers

```go
// pkg/controller/deployment/deployment_controller.go:117-146
dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    func(obj interface{}) { dc.addDeployment(logger, obj) },
    UpdateFunc: func(oldObj, newObj interface{}) { dc.updateDeployment(logger, oldObj, newObj) },
    DeleteFunc: func(obj interface{}) { dc.deleteDeployment(logger, obj) },
})
```

### Cache Sync Gate

```go
// pkg/controller/deployment/deployment_controller.go:189
if !cache.WaitForNamedCacheSyncWithContext(ctx, dc.dListerSynced, dc.rsListerSynced, dc.podListerSynced) {
    return
}
```

---

## 3. Workqueue: Typed Rate-Limiting Queue

**Source:** `staging/src/k8s.io/client-go/util/workqueue/queue.go` (lines 33–370), `rate_limiting_queue.go`

### What it does
A concurrent-safe work queue with three critical properties:
1. **Deduplication** (dirty set) — same item added twice results in one processing
2. **Re-entrancy** (processing set) — if an item is added while being processed, it's re-queued after Done()
3. **Rate limiting** — exponential backoff on failures

### Why
In a controller, multiple events may fire for the same object in rapid succession. Without deduplication, you'd process stale intermediate states. The dirty/processing set design ensures you always process the latest state while never losing notifications.

### The Dirty/Processing Dance

```go
// staging/src/k8s.io/client-go/util/workqueue/queue.go:227-252
func (q *Typed[T]) Add(item T) {
    q.cond.L.Lock()
    defer q.cond.L.Unlock()
    if q.shuttingDown { return }
    if q.dirty.Has(item) {
        if !q.processing.Has(item) {
            q.queue.Touch(item)  // Allow priority changes
        }
        return  // Already marked for processing
    }
    q.dirty.Insert(item)
    if q.processing.Has(item) {
        return  // Being processed, will re-queue on Done()
    }
    q.queue.Push(item)
    q.cond.Signal()
}

func (q *Typed[T]) Done(item T) {
    q.cond.L.Lock()
    defer q.cond.L.Unlock()
    q.processing.Delete(item)
    if q.dirty.Has(item) {
        q.queue.Push(item)  // Was modified during processing
        q.cond.Signal()
    }
}
```

### Rate-Limited Requeue

```go
// staging/src/k8s.io/client-go/util/workqueue/rate_limiting_queue.go:120-122
func (q *rateLimitingType[T]) AddRateLimited(item T) {
    q.TypedDelayingInterface.AddAfter(item, q.rateLimiter.When(item))
}
```

---

## 4. Tombstone Pattern (DeletedFinalStateUnknown)

**Source:** `staging/src/k8s.io/client-go/tools/cache/delta_fifo.go` (lines 797–801)

### What it does
When a watch disconnects and reconnects, some delete events may be missed. The DeltaFIFO synthesizes a `DeletedFinalStateUnknown` ("tombstone") containing the last known state of the object.

### Why
Without this, controllers would never learn about deletions that happened during disconnects.

```go
// staging/src/k8s.io/client-go/tools/cache/delta_fifo.go:797-801
type DeletedFinalStateUnknown struct {
    Key string
    Obj interface{}
}
```

### How controllers handle it

```go
// pkg/controller/deployment/deployment_controller.go:210-224
func (dc *DeploymentController) deleteDeployment(logger klog.Logger, obj interface{}) {
    d, ok := obj.(*apps.Deployment)
    if !ok {
        tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
        if !ok {
            utilruntime.HandleError(fmt.Errorf("couldn't get object from tombstone %#v", obj))
            return
        }
        d, ok = tombstone.Obj.(*apps.Deployment)
        if !ok {
            utilruntime.HandleError(fmt.Errorf("tombstone contained object that is not a Deployment %#v", obj))
            return
        }
    }
    dc.enqueueDeployment(d)
}
```

---

## 5. Controller Expectations Pattern

**Source:** `pkg/controller/controller_utils.go` (lines 130–315)

### What it does
Expectations track pending creates/deletes to prevent controllers from taking action on stale cache state. A controller won't sync until its expectations are satisfied or expired.

### Why
Between a controller issuing a create and the informer cache reflecting that new object, there's a window where the controller might create duplicates. Expectations close this gap.

```go
// pkg/controller/controller_utils.go:153-173
type ControllerExpectationsInterface interface {
    GetExpectations(controllerKey string) (*ControlleeExpectations, bool, error)
    SatisfiedExpectations(logger klog.Logger, controllerKey string) bool
    DeleteExpectations(logger klog.Logger, controllerKey string)
    SetExpectations(logger klog.Logger, controllerKey string, add, del int) error
    ExpectCreations(logger klog.Logger, controllerKey string, adds int) error
    ExpectDeletions(logger klog.Logger, controllerKey string, dels int) error
    CreationObserved(logger klog.Logger, controllerKey string)
    DeletionObserved(logger klog.Logger, controllerKey string)
}
```

---

## 6. OwnerReference / Controller Ref Manager Pattern

**Source:** `pkg/controller/controller_ref_manager.go` (lines 37–80)

### What it does
Implements garbage collection ownership through OwnerReferences. The ControllerRefManager handles adopting orphaned resources and releasing resources that no longer match.

### Why
Multiple controllers may create children. The ownership model ensures exactly one controller owns each child, enabling garbage collection and preventing conflicts.

```go
// pkg/controller/controller_ref_manager.go:37-50
type BaseControllerRefManager struct {
    Controller   metav1.Object
    Selector     labels.Selector
    canAdoptErr  error
    canAdoptOnce sync.Once      // Lazy, one-shot adoption check
    CanAdoptFunc func(ctx context.Context) error
}

// The claim logic: adopt if matching and orphaned, release if owned but not matching
func (m *BaseControllerRefManager) ClaimObject(ctx context.Context, obj metav1.Object,
    match func(metav1.Object) bool,
    adopt, release func(context.Context, metav1.Object) error) (bool, error) {
    controllerRef := metav1.GetControllerOfNoCopy(obj)
    if controllerRef != nil {
        if controllerRef.UID != m.Controller.GetUID() {
            return false, nil  // Owned by someone else
        }
        if match(obj) {
            return true, nil   // Already ours and matches
        }
        // Ours but no longer matches → release
    }
    // Orphan → adopt if matches
}
```

---

## 7. Leader Election Pattern

**Source:** `staging/src/k8s.io/client-go/tools/leaderelection/leaderelection.go` (lines 116–230)

### What it does
Provides distributed mutex semantics using a Kubernetes resource (Lease) as the lock. Only one instance of a controller runs actively; others are hot standbys.

### Why
Controller-manager runs multiple replicas for HA. Only one should reconcile to avoid conflicts.

```go
// staging/src/k8s.io/client-go/tools/leaderelection/leaderelection.go:116-163
type LeaderElectionConfig struct {
    Lock           rl.Interface
    LeaseDuration  time.Duration  // Default 15s — how long a lease is valid
    RenewDeadline  time.Duration  // Default 10s — how long leader retries renewal
    RetryPeriod    time.Duration  // Default 2s — how often candidates check
    Callbacks      LeaderCallbacks
    ReleaseOnCancel bool
}

type LeaderCallbacks struct {
    OnStartedLeading func(context.Context)
    OnStoppedLeading func()
    OnNewLeader      func(identity string)
}

// staging/src/k8s.io/client-go/tools/leaderelection/leaderelection.go:211-226
func (le *LeaderElector) Run(ctx context.Context) {
    defer runtime.HandleCrashWithContext(ctx)
    defer le.config.Callbacks.OnStoppedLeading()
    if !le.acquire(ctx) {
        return
    }
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    go le.config.Callbacks.OnStartedLeading(ctx)
    le.renew(ctx)
}
```

---

## 8. Feature Gates Pattern

**Source:** `pkg/features/kube_features.go` (lines 34–2811), `staging/src/k8s.io/client-go/features/features.go` (lines 34–80)

### What it does
A global registry of boolean flags that control feature rollout. Features progress through Alpha → Beta → GA → Deprecated lifecycle stages.

### Why
Kubernetes serves thousands of clusters. Features must be safe to enable/disable at runtime across versions. Feature gates provide:
- Progressive rollout (alpha off by default, beta on, GA locked)
- Per-version semantics (a feature may become beta in v1.28)
- Testing isolation

```go
// staging/src/k8s.io/client-go/features/features.go:50-70
type Feature string

type FeatureSpec struct {
    Default    bool
    LockToDefault bool
    PreRelease prerelease
    Version    *version.Version
}

type Gates interface {
    Enabled(key Feature) bool
}

// pkg/features/kube_features.go:50-58 (example feature definition)
const (
    AllowDNSOnlyNodeCSR featuregate.Feature = "AllowDNSOnlyNodeCSR"
    // ... 2700+ lines of feature definitions
)
```

### Registration at init()

```go
// pkg/features/kube_features.go:2798-2811
func init() {
    ca := &clientAdapter{utilfeature.DefaultMutableFeatureGate}
    runtime.Must(clientfeatures.AddVersionedFeaturesToExistingFeatureGates(ca))
    clientfeatures.ReplaceFeatureGates(ca)
    runtime.Must(utilfeature.DefaultMutableFeatureGate.AddVersioned(defaultVersionedKubernetesFeatureGates))
}
```
