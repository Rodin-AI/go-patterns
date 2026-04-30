# Anti-Patterns: What Kubernetes Avoids (and Why)

## 1. Never Mutate Shared Cache Objects

**What they avoid:** Modifying objects returned by Listers/Informers without deep-copying first.

**Why:** The informer cache is shared across all controllers. Mutating a cached object corrupts state for every other consumer.

**The pattern K8s enforces:**
```go
// WRONG — mutates shared cache
deployment, _ := dc.dLister.Deployments(ns).Get(name)
deployment.Spec.Replicas = ptr.To[int32](3)  // CORRUPTION!

// RIGHT — deep copy before mutating
deployment, _ := dc.dLister.Deployments(ns).Get(name)
deploymentCopy := deployment.DeepCopy()
deploymentCopy.Spec.Replicas = ptr.To[int32](3)
```

**Evidence:** The `runtime.Object` interface *mandates* `DeepCopyObject()`. Every API type has generated deep copy methods. The entire architecture assumes immutable reads.

---

## 2. Never Process the Same Key Concurrently

**What they avoid:** Multiple goroutines syncing the same object simultaneously.

**Why:** Two goroutines reading the same Deployment, each computing a different desired state, then both writing → conflict errors and potential state corruption.

**The pattern K8s enforces:** The workqueue's `processing` set ensures a key is only handed to one worker at a time. If an item is added while being processed, it's re-queued *after* `Done()` is called:

```go
// From queue.go — the processing set blocks concurrent access
func (q *Typed[T]) Get() (item T, shutdown bool) {
    // ... 
    q.processing.Insert(item)  // Mark as being worked on
    q.dirty.Delete(item)
    return item, false
}
```

---

## 3. Never Use Edge-Triggered Logic

**What they avoid:** Controllers that react to *what changed* rather than *what the current state is*.

**Why:** Events can be missed (watch disconnects), delivered out of order, or duplicated. If your controller says "a pod was deleted, so decrement counter" rather than "count current pods and compare to desired", you'll drift.

**The pattern K8s enforces:** Level-triggered reconciliation. The `syncHandler` reads *current state from the cache*, computes *desired state from the spec*, and makes the world match:

```go
// The sync function always reads current state, never relies on "what happened"
func (dc *DeploymentController) syncDeployment(ctx context.Context, key string) error {
    deployment, err := dc.dLister.Deployments(namespace).Get(name)
    // Compute desired state from deployment.Spec
    // Read actual state from replicaset lister
    // Reconcile difference
}
```

---

## 4. Never Forget to Call queue.Forget() on Success

**What they avoid:** Letting the rate limiter track items that succeeded.

**Why:** The rate limiter is per-item. If you never call `Forget()`, the next time the same key needs processing (even for a new event), it will be rate-limited as if it previously failed.

**Source:** The rate_limiting_queue.go comments explicitly warn:
```go
// NewTypedRateLimitingQueue constructs a new workqueue with rateLimited queuing ability
// Remember to call Forget!  If you don't, you may end up tracking failures forever.
```

**The pattern K8s enforces:**
```go
func (dc *DeploymentController) handleErr(ctx context.Context, err error, key string) {
    if err == nil {
        dc.queue.Forget(key)  // CRITICAL: clear the failure counter
        return
    }
    if dc.queue.NumRequeues(key) < maxRetries {
        dc.queue.AddRateLimited(key)
        return
    }
    dc.queue.Forget(key)  // Also forget when giving up
}
```

---

## 5. Never Hit the API Server in a Tight Loop

**What they avoid:** Direct API calls for reads. List/Get calls in hot paths.

**Why:** The API server is a shared resource. If 100 controllers each make 10 API calls per reconciliation at 1 sync/second, that's 1000 req/s to the API server per controller manager instance.

**The pattern K8s enforces:** Read from Listers (local cache), write to API server:
```go
// READ from cache (free, local, fast)
deployment, err := dc.dLister.Deployments(namespace).Get(name)

// WRITE to API server (expensive, remote, rate-limited)
_, err = dc.client.AppsV1().Deployments(namespace).Update(ctx, deployment, ...)
```

---

## 6. Never Sync Before Caches Are Warm

**What they avoid:** Processing items before the informer has done its initial List.

**Why:** With an empty cache, a controller might think "no pods exist, must create all of them" — causing a thundering herd of duplicate creates.

**The pattern K8s enforces:**
```go
// pkg/controller/deployment/deployment_controller.go:189
if !cache.WaitForNamedCacheSyncWithContext(ctx, 
    dc.dListerSynced, dc.rsListerSynced, dc.podListerSynced) {
    return  // Don't start workers until all caches are populated
}
```

---

## 7. Never Ignore Tombstones in Delete Handlers

**What they avoid:** Assuming delete handlers always receive the concrete type.

**Why:** If a watch disconnects and reconnects, missed deletes arrive as `DeletedFinalStateUnknown` (tombstones). Ignoring them means your controller never learns about those deletions.

**The pattern K8s enforces:**
```go
// Every delete handler must check for tombstones
func (dc *DeploymentController) deleteDeployment(logger klog.Logger, obj interface{}) {
    d, ok := obj.(*apps.Deployment)
    if !ok {
        tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
        if !ok {
            utilruntime.HandleError(...)
            return
        }
        d, ok = tombstone.Obj.(*apps.Deployment)
        // ...
    }
}
```

---

## 8. Never Use ResourceVersion for Equality

**What they avoid:** Comparing ResourceVersion to check if an object changed.

**Why:** ResourceVersion is opaque (currently etcd's mod_revision, but this is an implementation detail). The only valid operation is "!=" to detect change.

**The pattern K8s uses:**
```go
// pkg/controller/deployment/deployment_controller.go:284-288
func (dc *DeploymentController) updateReplicaSet(logger klog.Logger, old, cur interface{}) {
    curRS := cur.(*apps.ReplicaSet)
    oldRS := old.(*apps.ReplicaSet)
    if curRS.ResourceVersion == oldRS.ResourceVersion {
        return  // Periodic resync, nothing actually changed
    }
    // ... process the real update
}
```

---

## 9. Never Panic in Production Goroutines (Without Recovery)

**What they avoid:** Unhandled panics killing the entire controller manager.

**Why:** A single nil pointer in one controller's sync loop would crash all 30+ controllers running in the same process.

**The pattern K8s enforces:**
```go
// Every goroutine gets crash protection
func (dc *DeploymentController) Run(ctx context.Context, workers int) {
    defer utilruntime.HandleCrash()  // Top-level recovery
    // ...
}

// And in every polling loop:
func BackoffUntilWithContext(ctx context.Context, f func(ctx context.Context), ...) {
    func() {
        defer runtime.HandleCrashWithContext(ctx)  // Per-iteration recovery
        f(ctx)
    }()
}
```

---

## 10. Never Block Workers Indefinitely

**What they avoid:** Unbounded blocking in a sync handler (e.g., waiting for a condition that may never occur).

**Why:** Workers are a finite pool. If one blocks forever, that's one fewer worker processing the queue. At scale, this cascades.

**The pattern K8s enforces:** All API calls take context (with timeouts), all waits are bounded:
```go
// Timeouts on API calls
ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
defer cancel()
_, err := client.CoreV1().Pods(ns).Create(ctx, pod, metav1.CreateOptions{})
```

---

## 11. Never Use sync.Mutex Where sync.Once Suffices

**What they avoid:** Full mutual exclusion for one-shot operations.

**Why:** `sync.Once` is semantically clearer and avoids the bug where you forget to check a "done" flag under the mutex.

**The pattern K8s uses:**
```go
// pkg/controller/controller_ref_manager.go:43-49
type BaseControllerRefManager struct {
    canAdoptErr  error
    canAdoptOnce sync.Once      // One-shot lazy evaluation
    CanAdoptFunc func(ctx context.Context) error
}

func (m *BaseControllerRefManager) CanAdopt(ctx context.Context) error {
    m.canAdoptOnce.Do(func() {
        if m.CanAdoptFunc != nil {
            m.canAdoptErr = m.CanAdoptFunc(ctx)
        }
    })
    return m.canAdoptErr
}
```

---

## 12. Never Expose Mutable State Through Interfaces

**What they avoid:** Returning pointers to internal state through public interfaces.

**Why:** Callers can accidentally mutate internal state, creating subtle bugs that only manifest under concurrency.

**The pattern K8s enforces:** Listers return objects from the read-only cache. The `DeepCopy()` pattern ensures mutation safety is the caller's responsibility, not the cache's.

---

## Summary: The Philosophy

Kubernetes avoids these anti-patterns because of one fundamental truth: **in a distributed system, every assumption you make about state being consistent is wrong.**

The patterns exist because:
1. **Events are unreliable** → level-triggered reconciliation
2. **Reads are stale** → always compare desired vs actual
3. **Concurrent access is inevitable** → deep copy, queue serialization
4. **Failures are normal** → retry with backoff, graceful degradation
5. **Resources are shared** → cache reads, rate-limit writes
6. **Systems outlive their authors** → code generation, type registries, feature gates
