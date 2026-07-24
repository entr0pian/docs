# Kubernetes Operators

## Chapter 1: From `kubectl apply` to Running Pods

This chapter traces the full lifecycle of applying a custom resource — using a `Backend` operator as the example — from the moment you run `kubectl apply` to the moment the pods are running and status is updated.

---

### 1.1 The Operator Process Starts

When the operator pod boots, `main.go` runs in sequence:

1. **Scheme registration** — Go types are registered so the client knows how to serialize/deserialize objects. Both built-in Kubernetes types and your custom `Backend` type must be registered.

2. **Manager creation** — `ctrl.NewManager()` reads the cluster credentials (kubeconfig or in-cluster service account) and creates the shared infrastructure: the API client, the shared informer cache, the metrics server, and the health endpoints.

3. **Controller setup** — `SetupWithManager(mgr)` wires up the `BackendReconciler`. Internally this calls:

    ```go
    ctrl.NewControllerManagedBy(mgr).
        For(&appsv1alpha1.Backend{}).
        Owns(&appsv1.Deployment{}).
        Owns(&corev1.Service{}).
        Complete(r)
    ```

    `Complete(r)` is where the work queue is created, informers are registered, event handlers are attached, and worker goroutines are prepared.

4. **Manager start** — `mgr.Start()` is the blocking call that kicks everything off.

---

### 1.2 Informers: LIST then WATCH

When the manager starts, each registered informer goes through two phases.

**Phase 1 — LIST (one-time snapshot)**

The informer sends a LIST request to the API server:

```
GET /apis/apps.taskapp.io/v1alpha1/namespaces/default/backends
```

The API server reads all existing `Backend` objects from etcd and returns them. The informer writes every object into the **in-memory Store (cache)**. The response also includes a `resourceVersion` — a token representing the exact point in time of this snapshot.

**Phase 2 — WATCH (streaming, forever)**

The informer opens a long-lived HTTP connection:

```
GET /apis/apps.taskapp.io/v1alpha1/namespaces/default/backends?watch=true&resourceVersion=12345
```

Passing `resourceVersion` ensures no events are missed between the LIST and the WATCH. The API server streams events as they happen:

```json
{"type": "ADDED",    "object": {...}}
{"type": "MODIFIED", "object": {...}}
{"type": "DELETED",  "object": {...}}
```

The Reflector (inside the informer) reads this stream continuously.

**On every event:**
1. The **Store (cache) is updated** — always, unconditionally, before anything else.
2. The **event handler fires** — checks whether anyone cares about this event. For `Owns()` resources (e.g. Deployments), it checks `ownerReferences` and maps the owned object back to its parent `Backend` key. If owned → enqueue. If not → discard.

---

### 1.3 The Shared Informer Factory

Within a single operator process, the `SharedInformerFactory` ensures there is only **one** informer per resource type, regardless of how many controllers care about it.

```
Operator process
└── SharedInformerFactory
        ├── Informer for Backend    ← one watch stream, one cache
        ├── Informer for Deployment ← one watch stream, one cache
        └── Informer for Service    ← one watch stream, one cache
```

If two controllers in the same process both watch Deployments, they share the same stream and cache. Each controller registers its own event handler on the shared informer — the stream feeds all of them.

Note: the built-in Deployment controller runs in `kube-controller-manager`, a separate process. It has its own informer and watch stream. Sharing only applies within a process.

---

### 1.4 The Work Queue

The work queue sits between the informer event handlers and the `Reconcile` function. It provides:

- **Deduplication** — 10 events for the same object before the controller processes any of them results in one reconcile run, not 10.
- **Rate limiting** — prevents thundering herds after a cascade of changes.
- **Retry with backoff** — if `Reconcile` returns an error, the key is re-queued with exponential backoff.

The queue holds keys in `namespace/name` form — not full objects.

---

### 1.5 The Worker Goroutine

Between the queue and `Reconcile`, controller-runtime runs a worker goroutine:

```go
for {
    key, quit := queue.Get()       // blocks until something arrives
    err := r.Reconcile(ctx, key)
    if err != nil {
        queue.AddRateLimited(key)  // retry with backoff
    } else {
        queue.Forget(key)
    }
    queue.Done(key)
}
```

Multiple workers can run in parallel (`MaxConcurrentReconciles`). Each worker independently pops keys and calls `Reconcile`.

---

### 1.6 The Reconcile Signature

Every controller in controller-runtime must implement the `Reconciler` interface:

```go
type Reconciler interface {
    Reconcile(context.Context, Request) (Result, error)
}
```

This is a fixed contract — the signature never changes regardless of what the controller manages. The `Request` carries only the `namespace/name` key of the object that needs reconciling, not the event or the object itself. The two return values tell the work queue what to do next.

`ctrl.Result` has two fields:

```go
type Result struct {
    Requeue      bool
    RequeueAfter time.Duration
}
```

The zero value `ctrl.Result{}` means both fields are false/zero — do nothing, wait for the next watch event. The other forms used in the backend controller:

```go
ctrl.Result{}                            // done, no requeue
ctrl.Result{RequeueAfter: 15 * time.Second} // come back in 15s
ctrl.Result{Requeue: true}               // requeue immediately
```

---

### 1.7 How the Worker Interprets the Return Values

The worker uses the combination of `ctrl.Result` and `error` to decide what happens to the key after each reconcile run:

| `ctrl.Result` | `error` | Worker action |
|---|---|---|
| `{}` | `nil` | Key forgotten — wait for the next watch event |
| `{RequeueAfter: 15s}` | `nil` | Key re-added to the queue after 15 seconds |
| `{Requeue: true}` | `nil` | Key re-added to the queue immediately |
| anything | `non-nil` | `Result` is ignored — key re-added with exponential backoff |

The last row is important: **when an error is returned, `ctrl.Result` is ignored entirely**. The work queue owns the retry logic and applies backoff automatically. This is why error returns in the controller always pair with `ctrl.Result{}` — the value is irrelevant, it is just filling the slot:

```go
if err := r.reconcileDeployment(ctx, backend, queueURL); err != nil {
    return ctrl.Result{}, err   // Result is ignored; error drives the retry
}
```

In the deletion path specifically:

```go
if !backend.DeletionTimestamp.IsZero() {
    return ctrl.Result{}, r.handleDeletion(ctx, backend)
}
```

If `handleDeletion` returns `nil`, the key is forgotten — no explicit requeue needed. The finalizer removal inside `handleDeletion` triggers a MODIFIED watch event, which naturally re-enqueues the key one last time. That final reconcile hits `NotFound` and returns immediately. The loop goes quiet on its own.

If `handleDeletion` returns an error (e.g. the Deployment delete failed transiently), the key is re-queued with backoff and the deletion is retried automatically.

---

### 1.8 Reconcile: Desired vs Actual

`Reconcile` receives only a `namespace/name` key — not the event, not what changed. It always re-reads current state:

```go
func (r *BackendReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    backend := &appsv1alpha1.Backend{}
    r.Get(ctx, req.NamespacedName, backend)  // reads from cache, not API server

    r.reconcileDeployment(ctx, backend)      // make actual Deployment match spec
    r.reconcileService(ctx, backend)         // make actual Service match spec
    r.updateStatus(ctx, backend)             // report what actually exists
}
```

Inside `reconcileDeployment`:

```go
err := r.Get(ctx, key, existing)
if errors.IsNotFound(err) {
    return r.Create(ctx, desired)       // not there → create it
}
existing.Spec.Replicas = desired.Spec.Replicas
return r.Update(ctx, existing)          // there → update to match
```

This is **level-triggered** reconciliation: every run is a full "make actual = desired" pass. It doesn't matter how many events were missed or collapsed — the result is always convergence to the correct state.

---

### 1.9 The Convergence Loop

A single reconcile is rarely the last one. Each write to the API server generates new watch events, which trigger further reconcile runs, until there is nothing left to change.

```
kubectl apply -f backend.yaml
    → API server validates, stores in etcd, increments generation
    → ADDED event on watch stream
    → cache updated → "default/my-backend" enqueued

Reconcile #1
    → r.Get(backend) from cache
    → Deployment not found → r.Create(deployment) → API server → etcd
    → Service not found → r.Create(service) → API server → etcd
    → status updated: ReadyReplicas=0

    → Deployment ADDED event fires
    → Owns() handler maps it to Backend → enqueued again

Reconcile #2
    → Deployment exists, pods starting → ReadyReplicas=0
    → status updated (no change)

    → Deployment status changes as pods become ready
    → MODIFIED event → enqueued again

Reconcile #3
    → ReadyReplicas=1 → status updated: Available condition = True

    → no more writes → no more events → loop goes quiet
```

The loop reaches steady state naturally when `Reconcile` produces no more writes.

---

### 1.10 Owner References and Self-Healing

When `reconcileDeployment` creates the Deployment, it calls:

```go
ctrl.SetControllerReference(backend, deployment, r.Scheme)
```

This writes a `metadata.ownerReferences` entry on the Deployment pointing back to the Backend. Two consequences:

1. **Garbage collection** — when the Backend is deleted, Kubernetes automatically deletes the Deployment and Service. No cleanup code needed.
2. **Self-healing** — if someone manually deletes or edits the Deployment, the `Owns()` watch fires, maps it back to the parent Backend, enqueues it, and `Reconcile` restores the Deployment to match spec.

---

## Chapter 2: The CRD and Its Role

### 2.1 What the CRD Does

Before the operator is deployed, the cluster has no knowledge of the `Backend` type. The **CustomResourceDefinition (CRD)** is a separate manifest applied to the cluster that tells the API server:

- A new resource type `backends` exists under group `apps.taskapp.io`, version `v1alpha1`
- The API endpoint for it is `/apis/apps.taskapp.io/v1alpha1/namespaces/{ns}/backends`
- Here is the full OpenAPI schema: what fields exist, their types, which are required, what defaults apply
- `kubectl get backends` and `kubectl apply -f backend.yaml` are now valid operations

Once the CRD is applied, the API server stores and serves `Backend` objects. The controller is not required for this — the resource exists independently of whether anything is reconciling it.

---

### 2.2 The CRD is Generated, Not Hand-Written

The CRD YAML is generated automatically by `controller-gen` from the marker comments in your Go types:

```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Image",type=string,JSONPath=`.spec.image`
// +kubebuilder:printcolumn:name="Ready",type=integer,JSONPath=`.status.readyReplicas`
type Backend struct { ... }
```

Running `make generate manifests` produces `config/crd/bases/apps.taskapp.io_backends.yaml`. The schema in that file is derived directly from the Go struct field types and validation tags — the Go code is the single source of truth.

---

### 2.3 What the CRD File Contains

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backends.apps.taskapp.io
spec:
  group: apps.taskapp.io
  names:
    kind: Backend
    plural: backends
    singular: backend
  scope: Namespaced
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:        # full field validation schema
        ...
    subresources:
      status: {}              # status is a separate subresource
    additionalPrinterColumns: # columns shown by kubectl get backends
      ...
```

Key fields:
- `group` + `names` — defines the API endpoint path
- `scope: Namespaced` — objects live in a namespace (vs `Cluster`-scoped)
- `openAPIV3Schema` — the API server validates every apply against this before storing anything
- `subresources.status` — makes status a separate endpoint; `r.Status().Update()` uses `/status` and cannot accidentally overwrite spec

---

### 2.4 Validation Happens at the API Server, Before the Controller

When you run `kubectl apply -f backend.yaml`, the API server validates the manifest against the CRD's OpenAPI schema **before** it is stored in etcd and **before** the controller ever sees it. A bad manifest is rejected immediately with a clear error — the reconcile loop is never involved.

This is why the CRD is the right place to express constraints: required fields, allowed values, minimum/maximum — not `if` statements inside `Reconcile`.

---

### 2.5 Deployment Order

The CRD must exist before the controller is deployed. If the controller starts and tries to LIST a resource type the API server doesn't know about, it crashes.

```
Step 1: Apply CRD
    → API server registers the backends endpoint
    → kubectl get backends  → returns empty list (works)
    → kubectl apply -f backend.yaml → stored, not yet reconciled

Step 2: Deploy the operator
    → informer LISTs /apis/apps.taskapp.io/v1alpha1/backends  (endpoint exists)
    → WATCH stream opens
    → any Backend objects from Step 1 are picked up and reconciled
```

In a GitOps setup (ArgoCD), the CRD is typically applied in an earlier sync wave (wave 0 or 1) than the controller deployment (wave 2) to enforce this order automatically.

---

### 2.6 The CRD Enables Native Kubernetes Behaviour

Because `Backend` is a real API resource (not a ConfigMap or annotation hack), it gets all native Kubernetes features for free:

- **RBAC** — `kubectl create role ... --resource=backends.apps.taskapp.io`
- **`kubectl wait`** — `kubectl wait backend/my-backend --for=condition=Available`
- **Events** — `kubectl describe backend my-backend` shows controller events
- **`kubectl get` columns** — defined via `additionalPrinterColumns` in the CRD
- **`resourceVersion` and optimistic locking** — the API server manages this automatically
- **Garbage collection** — ownerReferences work because the API server understands the type

---

## Chapter 3: resourceVersion and Optimistic Concurrency

### 3.1 What resourceVersion Is

Every object stored in etcd has a `resourceVersion` field stamped on it by the API server. It is a monotonically increasing integer that represents the **etcd revision at which the object was last written**.

```yaml
metadata:
  name: crd-backend
  resourceVersion: "12723"
```

Your controller never sets this field. It is assigned by etcd on every write and returned to you on every read. You only ever read it — either to know the current version of an object, or to pass it back as a precondition on an update.

---

### 3.2 The Four Roles of resourceVersion

The same field serves four distinct purposes, all based on the same underlying etcd revision counter:

| Context | Role |
|---|---|
| On an individual object | "This is the etcd revision at which this object was last written" |
| On a LIST response | "This is the etcd revision of this entire collection snapshot" |
| On a WATCH request | "Only stream events that happened after this revision" |
| On an UPDATE request | "Reject my write if the object changed since I last read it" |

---

### 3.3 resourceVersion in the Informer Lifecycle

When the manager starts, each informer runs the LIST → WATCH sequence. `resourceVersion` is the link between the two phases:

```
Informer boots
    ↓
LIST /apis/apps.taskapp.io/v1alpha1/backends
    ← response: all Backend objects + list-level resourceVersion: 12500
    ↓
    (cache populated with everything up to revision 12500)
    ↓
WATCH /apis/apps.taskapp.io/v1alpha1/backends?watch=true&resourceVersion=12500
    ← streams only events that happened after revision 12500
```

Passing the list-level `resourceVersion` to the WATCH call guarantees no events are missed in the gap between LIST completing and WATCH starting. The API server picks up exactly where the snapshot left off.

As events arrive, the informer continuously updates its internal bookmark to the latest `resourceVersion` it has seen. If the WATCH connection drops and reconnects, it resumes from that bookmark — not from the beginning — so no events are replayed unnecessarily.

---

### 3.4 resourceVersion as the Optimistic Lock

Controllers are level-triggered and may run concurrently. Two reconcile workers — or two separate operator instances during a rollout — can read the same object and attempt to write it at the same time. The API server uses `resourceVersion` to detect and reject the losing write.

**Happy path — no concurrent writer:**

```
r.Get()    → Backend object, resourceVersion: 12500
             (modify fields in memory)
r.Update() → sends object with resourceVersion: 12500
             API server checks: 12500 == 12500 ✓
             etcd writes, assigns new revision: 12501
             API server returns object with resourceVersion: 12501
```

**Conflict path — concurrent writer wins first:**

```
r.Get()    → Backend object, resourceVersion: 12500
             [another writer updates the object → etcd now at 12600]
r.Update() → sends object with resourceVersion: 12500
             API server checks: 12500 ≠ 12600 ✗
             → 409 Conflict returned
```

The API server does not attempt to merge or retry. It simply rejects the stale write. Your controller never corrupts state because the check is atomic inside etcd — only one writer can win per revision.

---

### 3.5 How controller-runtime Handles a 409

Controller-runtime catches the 409 inside the client and surfaces it as an error back to `Reconcile`. The worker goroutine then re-enqueues the key with rate-limited backoff:

```
Reconcile() returns err (409 Conflict)
    ↓
work queue: AddRateLimited("default/crd")
    ↓
worker sleeps (backoff)
    ↓
Reconcile() runs again
    r.Get()    → fresh object with resourceVersion: 12600
    r.Update() → sends 12600 → accepted → etcd writes 12601
```

This is why the reconcile pattern is always **Get → modify in memory → Update**. You must never reuse a stale object pointer across reconcile runs — the embedded `resourceVersion` would be wrong and every update attempt would return a 409.

---

### 3.6 Validating the Absence of a No-Op Update Loop

A naive reconciler that calls `r.Update()` unconditionally on every run — even when nothing changed — generates a write on each reconcile. That write:

1. Bumps the `resourceVersion` in etcd
2. Fires a new MODIFIED watch event
3. Re-enqueues the controller via the `Owns()` handler
4. Triggers another reconcile → another update → repeat

The fix is a `DeepEqual` guard before the update call:

```go
if equality.Semantic.DeepEqual(existing.Spec.Replicas, desired.Spec.Replicas) &&
    equality.Semantic.DeepEqual(existing.Spec.Template.Spec, desired.Spec.Template.Spec) {
    return nil   // nothing changed — skip the write
}
existing.Spec.Replicas = desired.Spec.Replicas
existing.Spec.Template.Spec = desired.Spec.Template.Spec
return r.Update(ctx, existing)
```

No write → no watch event → no re-enqueue → loop goes quiet.

**How to verify this in practice:** sample the `resourceVersion` of the managed Deployment at intervals after the system reaches steady state. If the loop is gone, the value is stable:

```bash
kubectl get deployment crd-backend -o jsonpath='{.metadata.resourceVersion}'
# wait 5 seconds
kubectl get deployment crd-backend -o jsonpath='{.metadata.resourceVersion}'
# same value → no writes happening → loop is broken
```

A continuous stream of `Configured` events from `kubectl get events` is the other signal that a no-op loop is active.

---

## Chapter 4: generation and observedGeneration

### 4.1 metadata.generation

Every Kubernetes resource has a `metadata.generation` field — a monotonically increasing integer owned and incremented exclusively by the API server. It starts at `1` when the object is created and increments by one **every time the `.spec` changes**.

```
kubectl apply -f backend.yaml          → generation: 1  (creation)
kubectl patch backend ... image=v2     → generation: 2  (spec changed)
kubectl patch backend ... replicas=3   → generation: 3  (spec changed)
```

Changes to `.metadata` (labels, annotations) and `.status` do **not** bump generation — only spec changes do. This makes `generation` a reliable counter of how many times the desired state has evolved.

---

### 4.2 status.observedGeneration

`status.observedGeneration` is the controller's acknowledgement field. After a successful reconcile, the controller sets it to the `generation` value it processed:

```go
// backend_controller.go
cond := metav1.Condition{
    ObservedGeneration: backend.Generation,  // echo back what we just processed
    ...
}
```

The controller does not maintain its own counter — it simply copies `metadata.generation` into the status once reconciliation is complete. The API server owns the counter; the controller owns the echo.

---

### 4.3 Reading the Gap

The gap between the two fields is the observable signal of whether the controller is caught up:

```
metadata.generation:        4   ← API server: spec has changed 4 times
status.observedGeneration:  3   ← controller: last fully reconciled generation 3
```

→ The controller is still working on the latest spec change. Any status fields (readyReplicas, conditions) reflect generation 3, not the current spec.

```
metadata.generation:        4
status.observedGeneration:  4   ← caught up
```

→ The status reflects the current spec.

This is the mechanism behind `kubectl rollout status` — it watches until `observedGeneration` catches up to `generation` and `readyReplicas` reaches the desired count.

---

### 4.4 Observing It in Practice

```bash
kubectl get deployments -A --context kind-prod \
  -o custom-columns="NAME:.metadata.name,NAMESPACE:.metadata.namespace,GENERATION:.metadata.generation,OBSERVED:.status.observedGeneration"
```

For your own `Backend` CRs:

```bash
kubectl get backends -A --context kind-prod \
  -o custom-columns="NAME:.metadata.name,GENERATION:.metadata.generation,OBSERVED:.status.observedGeneration,READY:.status.readyReplicas"
```

When both columns show the same value for every resource, the system is fully converged.

---

### 4.5 Why It Matters

Without `observedGeneration`, a consumer watching `.status` cannot tell whether a stale status is from the current spec or a previous one. Consider this sequence:

1. `Backend` is at generation 3, `Available=True`, 2/2 replicas ready.
2. You change the image tag → generation becomes 4.
3. The rollout starts — pods are terminating and restarting.
4. For a few seconds `Available=True` is still in `.status`, reflecting generation 3.

With `observedGeneration`, a consumer can detect that `observedGeneration=3 < generation=4` and treat the status as stale while the controller works through the rollout. Without it, `Available=True` looks authoritative even though it describes the old spec.

---

## Chapter 5: Finalizers and Controlled Deletion

### 5.1 The Default Deletion Path — Garbage Collection

When an operator uses `ctrl.SetControllerReference` to set `ownerReferences` on child objects, deletion of the parent is normally handled by the **Kubernetes Garbage Collector (GC)** — a controller running inside `kube-controller-manager`.

The deletion flow without a finalizer:

```
kubectl delete backend my-backend
    ↓
API server removes object from etcd immediately
    ↓
GC controller watches for objects whose owner no longer exists
    ↓
GC deletes the orphaned Deployment and Service
```

The operator is never involved. By the time any watch event could fire, the Backend CR is already gone. The Deployment and Service disappear asynchronously, after a short delay, once the GC notices the orphaned `ownerReferences`.

This works correctly in a real cluster and is the simplest approach. The downside: cleanup is not instantaneous, it cannot be observed or retried by your controller, and it does not work in `envtest` (which does not run `kube-controller-manager`).

---

### 5.2 What a Finalizer Is

A finalizer is a string stored in `metadata.finalizers`:

```yaml
metadata:
  name: my-backend
  finalizers:
    - apps.taskapp.io/finalizer
```

When this list is non-empty, **Kubernetes refuses to delete the object from etcd**. Instead it:

1. Sets `metadata.deletionTimestamp` to the current time
2. Keeps the object alive in the API server
3. Continues delivering watch events to any controllers that have the object in their `For()` or `Owns()` registration

The object now has two observable states: normal (`deletionTimestamp` is nil) and terminating (`deletionTimestamp` is set). The controller is responsible for watching for the terminating state and acting on it.

---

### 5.3 The Deletion Path With a Finalizer

```
kubectl delete backend my-backend
    ↓
API server sets deletionTimestamp — object stays in etcd
    ↓
Watch event (MODIFIED) fires → "default/my-backend" enqueued
    ↓
Reconcile() runs
    → sees deletionTimestamp != nil → calls handleDeletion()
    → explicitly deletes Deployment
    → explicitly deletes Service
    → removes finalizer from metadata.finalizers
    → r.Update(backend)
    ↓
API server sees finalizers list is now empty
    ↓
API server deletes object from etcd
    ↓
Watch event (DELETED) fires → reconciler runs one last time
    → r.Get() returns NotFound → returns nil, nothing to do
```

The operator owns every step. The GC is not involved at all.

---

### 5.4 The Backend Operator Implementation

The finalizer is wired into `Reconcile` as a branch on `deletionTimestamp`:

```go
const backendFinalizer = "apps.taskapp.io/finalizer"

func (r *BackendReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    backend := &appsv1alpha1.Backend{}
    if err := r.Get(ctx, req.NamespacedName, backend); err != nil {
        if errors.IsNotFound(err) {
            return ctrl.Result{}, nil   // already gone
        }
        return ctrl.Result{}, err
    }

    // Terminating branch — run cleanup then remove the finalizer
    if !backend.DeletionTimestamp.IsZero() {
        return ctrl.Result{}, r.handleDeletion(ctx, backend)
    }

    // Normal branch — add finalizer on first reconcile, then reconcile children
    if !controllerutil.ContainsFinalizer(backend, backendFinalizer) {
        controllerutil.AddFinalizer(backend, backendFinalizer)
        return ctrl.Result{}, r.Update(ctx, backend)
    }

    // ... reconcileDeployment, reconcileService, updateStatus
}
```

`handleDeletion` explicitly deletes the owned resources and then removes the finalizer:

```go
func (r *BackendReconciler) handleDeletion(ctx context.Context, backend *appsv1alpha1.Backend) error {
    if !controllerutil.ContainsFinalizer(backend, backendFinalizer) {
        return nil
    }
    // delete Deployment, delete Service ...
    controllerutil.RemoveFinalizer(backend, backendFinalizer)
    return r.Update(ctx, backend)
}
```

---

### 5.5 The Extra Reconcile on Create

Adding the finalizer is a write to the API server. That write fires a MODIFIED watch event, which re-enqueues the Backend CR. So the first reconcile after a new Backend is created does nothing except add the finalizer:

```
Backend CR created → Reconcile #1
    → deletionTimestamp nil
    → finalizer not present → AddFinalizer → r.Update()
    → returns

MODIFIED event fires → Reconcile #2
    → deletionTimestamp nil
    → finalizer present → proceeds to reconcileDeployment, reconcileService
```

The Deployment and Service are created on the second reconcile, not the first. This is expected and harmless — the extra round trip is a few milliseconds in a real cluster.

---

### 5.6 Finalizer vs Owner Reference — When to Use Each

| | Owner reference + GC | Finalizer |
|---|---|---|
| Who deletes children | Kubernetes GC | Your controller |
| Works in `envtest` | No | Yes |
| Cleanup is synchronous | No (GC is async) | Yes (controller retries on error) |
| Code complexity | None | `handleDeletion` function |
| Cleanup can fail silently | Yes (GC has no feedback) | No (error re-queues and retries) |
| Object stays visible during cleanup | No (CR gone immediately) | Yes (`deletionTimestamp` set) |

Both approaches set `ownerReferences`. The difference is only in who acts on them. For most operators, owner references alone are sufficient. A finalizer is warranted when you need cleanup to be observable, retryable, or testable in `envtest`.

---

## Chapter 6: Deep Copy and the Cache Protection Problem

### 6.1 Why Deep Copy Exists

The operator's informer cache is a shared, in-memory store. Every time a watch event arrives from the API server, the cache is updated with the latest version of that object. All reconcile workers — potentially running in parallel — read from this same cache.

If `r.Get` returned a direct pointer into the cache, any modification you made to the object in memory would silently corrupt the cached version. The next reconcile worker to read that object would get a mutated copy that was never written to etcd, and the controller's view of the world would diverge from reality.

To prevent this, controller-runtime enforces a rule: **you never get a reference to the cached object**. Every `r.Get` call writes a deep copy of the cached object into your local variable. You get an independent clone — modifying it cannot affect the cache.

---

### 6.2 How `r.Get` Uses Deep Copy

When you call:

```go
backend := &appsv1alpha1.Backend{}
r.Get(ctx, req.NamespacedName, backend)
```

Your `backend` is an empty local variable — the destination. Internally, controller-runtime does roughly:

```go
cachedObject := informerCache.Get(name)   // the real cached object
cachedObject.DeepCopyInto(backend)        // cached → your local variable
```

`DeepCopyInto` is called **on the cached object**, writing into your `backend`. You never touch the cached object. The result is that `backend` is a fully independent copy — same values, different memory.

```
After r.Get():

cache.backend  ──► [ Backend { Spec: {Image: "v1", Replicas: *→[2]} } ]   ← untouched
your backend   ──► [ Backend { Spec: {Image: "v1", Replicas: *→[2]} } ]   ← independent copy
```

Modifying `backend.Spec.Image = "v2"` in your reconcile loop changes only your copy.

---

### 6.3 The `runtime.Object` Interface Requirement

For this to work generically — without controller-runtime knowing the concrete type at compile time — every Kubernetes object must implement the `runtime.Object` interface:

```go
type Object interface {
    GetObjectKind() schema.ObjectKind
    DeepCopyObject() runtime.Object
}
```

`DeepCopyObject()` is the method that allows the runtime to clone any object without knowing its concrete type. It returns a new `runtime.Object` that is a fully independent copy.

This is why `zz_generated.deepcopy.go` exists — it implements this interface for every type you define. The file is generated (not hand-written) by `controller-gen`, which reads your struct definitions and produces the correct copy logic for every field type automatically.

---

### 6.4 Why a Shallow Copy Is Not Enough

A shallow copy duplicates the struct's immediate values byte-for-byte. For primitive types (`string`, `int32`, `bool`) this is safe. For **reference types** — pointers, slices, and maps — it is not.

Consider `BackendSpec`:

```go
type BackendSpec struct {
    Image    string
    Tag      string
    Replicas *int32   // ← pointer
    DBSecret string
}
```

A shallow copy of this struct (`*out = *in`) produces:

```
in.Image     "boicotaz/taskapp-backend"      out.Image     "boicotaz/taskapp-backend"
in.Tag       "abc123"                        out.Tag       "abc123"
in.Replicas  ──────────────────────────────────────────────►  [ int32: 2 ]
```

`in.Replicas` and `out.Replicas` now point to the same `int32` in memory. If the reconcile loop writes `*out.Replicas = 5`, it also changes `*in.Replicas` — which is the value sitting in the cache. The cache now says 5 replicas when the user specified 2, and no write to etcd ever happened. The controller's view is silently corrupted.

The same problem applies to maps and slices, where both copies would share the same underlying array or hash table.

---

### 6.5 Deep Copy Function Analysis

Here is the generated `BackendSpec.DeepCopyInto` from `zz_generated.deepcopy.go`:

```go
func (in *BackendSpec) DeepCopyInto(out *BackendSpec) {
    *out = *in
    if in.Replicas != nil {
        in, out := &in.Replicas, &out.Replicas
        *out = new(int32)
        **out = **in
    }
}
```

**Line 1 — `*out = *in`**

Dereferences both pointers and copies the struct value. This is a shallow copy — all fields are copied byte-for-byte. Primitive fields (`Image`, `Tag`, `DBSecret`) are correctly duplicated since strings in Go are immutable (they can be reassigned but the underlying bytes cannot be mutated through the variable, so sharing is safe). `Replicas` is copied as a raw pointer value — both `in.Replicas` and `out.Replicas` now hold the same address. This will be fixed in the next block.

**Line 2 — `if in.Replicas != nil`**

Guard against nil pointers. If `Replicas` was not set, both `in.Replicas` and `out.Replicas` are `nil` after the shallow copy — safe, nothing to fix.

**Line 3 — `in, out := &in.Replicas, &out.Replicas`**

This shadows the outer `in` and `out` variables with new, narrower ones:
- `in` is now a `**int32` — a pointer to the `Replicas` field of the source struct
- `out` is now a `**int32` — a pointer to the `Replicas` field of the destination struct

This lets the next two lines operate directly on those fields without touching the surrounding struct.

**Line 4 — `*out = new(int32)`**

Allocates a brand new `int32` on the heap and stores its address in `out.Replicas`. The destination's `Replicas` field now points to fresh, unshared memory. The source's `Replicas` field is unchanged.

**Line 5 — `**out = **in`**

Dereferences `in` twice (pointer to pointer to int32 → the int32 value itself) and copies that integer value into the newly allocated memory. The result: both `in.Replicas` and `out.Replicas` point to different `int32` addresses that hold the same value.

```
Before fix (after *out = *in):
in.Replicas  ──┐
               ├──► [ int32: 2 ]   ← same address
out.Replicas ──┘

After fix:
in.Replicas  ──► [ int32: 2 ]   ← original allocation
out.Replicas ──► [ int32: 2 ]   ← new independent allocation
```

---

### 6.6 How Maps and Slices Are Handled

For types with maps or slices the generator produces more elaborate copy logic. A `map[string]string` (such as the `Tags` field planned for `SQSSpec`) would generate:

```go
func (in *SQSSpec) DeepCopyInto(out *SQSSpec) {
    *out = *in
    if in.Tags != nil {
        in, out := &in.Tags, &out.Tags
        *out = make(map[string]string, len(*in))
        for key, val := range *in {
            (*out)[key] = val
        }
    }
}
```

The same pattern: shallow copy first, then fix the reference type. `make(map[string]string, len(*in))` allocates a new map with the same capacity, then each key-value pair is copied individually. The two maps now occupy separate memory — inserting or deleting keys in one cannot affect the other.

A slice follows the same logic: `*out = make([]T, len(*in))` allocates a new backing array, then `copy(*out, *in)` fills it.

---

### 6.7 `DeepCopyObject` vs `DeepCopyInto`

Two methods exist for different use cases:

**`DeepCopyInto(out)`** — copies into a pre-allocated destination. Used when the caller already has an object to write into. This is what `r.Get` uses internally — your local `backend` variable is the pre-allocated destination.

**`DeepCopyObject()`** — allocates a new object and returns it as a `runtime.Object`. Used when the caller has no destination yet — for example when the informer receives a new object from the watch stream and needs to store an independent copy in the cache.

```go
// DeepCopyObject is implemented in terms of DeepCopyInto:
func (in *Backend) DeepCopyObject() runtime.Object {
    if in == nil {
        return nil
    }
    out := new(Backend)      // allocate fresh
    in.DeepCopyInto(out)     // copy into it
    return out               // return as interface
}
```

`DeepCopyInto` is the workhorse. `DeepCopyObject` is the interface adapter that the runtime calls when it only knows the type as `runtime.Object`.

---

### 6.8 Why `make generate` Must Be Re-Run After Struct Changes

`zz_generated.deepcopy.go` is derived from your type definitions. It is not maintained manually. When you add a new field — especially a reference type like a map, slice, or pointer — the generated file becomes stale. The old `DeepCopyInto` has no code to handle the new field, so the shallow copy (`*out = *in`) is the only thing that runs for it.

For the planned `SQSSpec` addition:

```go
type SQSSpec struct {
    QueueName string
    Region    string
    Tags      map[string]string   // ← new map field
}
```

Without re-running `make generate`, `SQSSpec.DeepCopyInto` would not exist at all (the type is new), and `BackendSpec.DeepCopyInto` would not call `in.SQS.DeepCopyInto(out.SQS)` for the new pointer field — both the `SQSSpec` struct and its `Tags` map would be shallow-copied. The `Tags` map would be shared between the cache and every reconcile copy, and mutations to tags in one reconcile run would corrupt the cache.

Running `make generate` reads the updated struct definitions and produces correct copy logic for every field, including the new map. The build fails to compile if `DeepCopyObject` is missing from any registered type, so the regeneration step cannot be silently skipped.

---

## Chapter 7: Kubebuilder Markers

### 7.1 What Markers Are

Markers look like comments but they are not. A normal Go comment is ignored by every tool except a human reader. A marker is a structured instruction that `controller-gen` — the code generator shipped with kubebuilder — reads when you run `make generate` or `make manifests`. At runtime they are completely invisible; the compiled binary has no trace of them. They only matter at generation time.

```go
// +kubebuilder:default=standard
// +optional
Type QueueType `json:"type,omitempty"`
```

The `//` prefix means the Go compiler discards them. The `+kubebuilder:` prefix is the signal `controller-gen` looks for. Everything after that prefix is the instruction.

---

### 7.2 Where the Output Goes

Markers on different targets produce output in different files:

| Marker location | Output file |
|---|---|
| On a `struct` type or its fields | `config/crd/bases/<group>_<plural>.yaml` — the CRD schema |
| On the `Reconcile` method (RBAC markers) | `config/rbac/role.yaml` — the ClusterRole for the operator |
| On the package (deep copy markers) | `api/v1alpha1/zz_generated.deepcopy.go` |

None of these files are hand-maintained. They are regenerated on every `make generate manifests` run. The Go source is the single source of truth; the generated files are artifacts.

---

### 7.3 Validation Markers

These translate directly into constraints in the CRD's `openAPIV3Schema`. The API server enforces them at admission — before the object is written to etcd, before the controller ever sees it.

**`// +kubebuilder:validation:Required`**

Adds the field name to the parent object's `required:` list in the schema. Any `Backend` CR that omits this field is rejected immediately with a clear validation error.

```go
// +kubebuilder:validation:Required
Image string `json:"image"`
```

Generated CRD:
```yaml
required:
  - image
  - tag
  - dbSecret
```

**`// +optional`**

The inverse — marks the field as not required. For pointer fields (`*QueueSpec`) this is implicit since a nil pointer serializes to absent, but writing it explicitly makes the intent clear to readers. For value types like `bool` it is necessary, because an omitted `bool` would otherwise default to `false` via Go's zero value without the schema knowing it was intentionally omitted.

**`// +kubebuilder:validation:Enum=standard;fifo`**

Adds an `enum:` constraint to the schema. Values outside the enumerated list are rejected at admission.

```go
// +kubebuilder:validation:Enum=standard;fifo
type QueueType string
```

Generated CRD:
```yaml
type:
  enum:
  - standard
  - fifo
  type: string
```

---

### 7.4 Default Markers

**`// +kubebuilder:default=<value>`**

Sets a server-side default in the schema. When the field is omitted from a submitted CR, the Kubernetes API server fills it in before storing the object. By the time the controller reads `backend.Spec.Queue.Type`, it is always `"standard"` — never an empty string — even if the user wrote only `queue: {}`.

```go
// +kubebuilder:default=standard
// +optional
Type QueueType `json:"type,omitempty"`
```

Generated CRD:
```yaml
type:
  default: standard
  enum:
  - standard
  - fifo
  type: string
```

This eliminates a whole class of defensive code. Without this marker you would need:

```go
if backend.Spec.Queue.Type == "" {
    backend.Spec.Queue.Type = "standard"
}
```

With the marker, the API server has already guaranteed the value is set, and that `if` block becomes unnecessary noise.

---

### 7.5 Object and Subresource Markers

These go on the type declaration itself, not on individual fields.

**`// +kubebuilder:object:root=true`**

Tells controller-gen that this type is a top-level API object (a CR that can be applied with `kubectl apply`), not just a nested struct like `QueueSpec`. It causes `controller-gen` to:
1. Generate a `DeepCopyObject()` method for the type (required by `runtime.Object`)
2. Register the type in the generated scheme

Without this marker, the type exists as a Go struct but the API server and controller-runtime have no way to work with it as a Kubernetes object.

**`// +kubebuilder:subresource:status`**

Splits the status into a separate API endpoint (`/status`). Two consequences:

1. `r.Status().Update()` (or `r.Status().Patch()`) writes only to the status subresource, not the whole object. This prevents a status update from accidentally overwriting spec changes that arrived between the controller's last `r.Get()` and the update call.
2. RBAC on the main resource and on `/status` can be granted separately. Application developers can be given write access to spec without being able to modify status.

Without this marker, status and spec share the same endpoint — a single `r.Update()` writes both, and a spec change racing with a status update could clobber one or the other.

---

### 7.6 Printer Column Markers

**`// +kubebuilder:printcolumn:name="...",type=...,JSONPath=...`**

Controls what `kubectl get backends` shows. Each marker adds one column. The `priority` field controls visibility: `priority=0` (default) is shown in normal output; `priority=1` requires `-o wide`.

```go
// +kubebuilder:printcolumn:name="Tag",type=string,JSONPath=`.spec.tag`
// +kubebuilder:printcolumn:name="Ready",type=integer,JSONPath=`.status.readyReplicas`
// +kubebuilder:printcolumn:name="QueueURL",type=string,JSONPath=`.status.queueURL`,priority=1
```

Generated CRD:
```yaml
additionalPrinterColumns:
- jsonPath: .spec.tag
  name: Tag
  type: string
- jsonPath: .status.readyReplicas
  name: Ready
  type: integer
- jsonPath: .status.queueURL
  name: QueueURL
  priority: 1
  type: string
```

The JSONPath expressions are evaluated by `kubectl` against the object, not by the API server. The API server stores them verbatim in the CRD; `kubectl` applies them when rendering the table.

---

### 7.7 RBAC Markers

**`// +kubebuilder:rbac:groups=...,resources=...,verbs=...`**

These go on the `Reconcile` method and generate the operator's ClusterRole in `config/rbac/role.yaml`.

```go
// +kubebuilder:rbac:groups=apps.taskapp.io,resources=backends,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=sqs.aws.upbound.io,resources=queues,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=sqs.aws.upbound.io,resources=queues/status,verbs=get
```

Generated `role.yaml`:
```yaml
- apiGroups: [apps.taskapp.io]
  resources: [backends]
  verbs: [get, list, watch, create, update, patch, delete]
- apiGroups: [sqs.aws.upbound.io]
  resources: [queues]
  verbs: [get, list, watch, create, update, patch, delete]
- apiGroups: [sqs.aws.upbound.io]
  resources: [queues/status]
  verbs: [get]
```

Notice the separate entry for `queues/status` with only `get`. The operator reads Queue status (to check the `Ready` condition and the `atProvider.url`) but never writes it — Crossplane owns the status subresource for its own CRDs. The RBAC markers express least-privilege directly at the source.

---

### 7.8 The Generation Pipeline

The full relationship between markers and generated artifacts:

```
Go source (markers in comments)
    ↓  make generate
zz_generated.deepcopy.go     ← DeepCopyInto / DeepCopyObject for every type
    ↓  make manifests
config/crd/bases/*.yaml      ← full CRD with OpenAPI schema, defaults, printer columns
config/rbac/role.yaml        ← ClusterRole with least-privilege RBAC rules
```

`make generate` must run first because `make manifests` reads the generated deepcopy file as part of type analysis. Both must run whenever the type definitions change — adding a field, changing a type, adding a validation constraint.

The generated files are committed to the repository. When ArgoCD syncs, it applies the CRD YAML from the repo. This means the CRD in the cluster always reflects exactly the markers in the Go source at the time of the last `make manifests` run — the cluster state is traceable back to the source code.

---

## Chapter 8: Resource Structure and Status

### 8.1 The Four Parts of Every Kubernetes Object

Every Kubernetes object — built-in or custom — is composed of four embedded pieces:

```go
type Backend struct {
    metav1.TypeMeta   `json:",inline"`        // kind + apiVersion
    metav1.ObjectMeta `json:"metadata,..."`   // name, namespace, labels, resourceVersion, ...
    Spec   BackendSpec   `json:"spec"`        // desired state — written by the user
    Status BackendStatus `json:"status,..."`  // observed state — written by the controller
}
```

**`metav1.TypeMeta`** holds `Kind` and `APIVersion`. This is how the API server knows what type it is dealing with when it deserializes a manifest. When you write `kind: Backend` in YAML, that value lands here.

**`metav1.ObjectMeta`** is the standard metadata block present on every Kubernetes object. It contains `Name`, `Namespace`, `Labels`, `Annotations`, `ResourceVersion`, `Generation`, `DeletionTimestamp`, `Finalizers`, `OwnerReferences`, and more. Everything covered in Chapters 3–5 lives inside this struct. By embedding it, the `Backend` type inherits all of that for free.

**`Spec`** is the desired state — what the user or CI declares. It is the input to the controller.

**`Status`** is the observed state — what the controller reports back after acting on the spec. It is the output of the controller. The user never writes it; the `omitzero` tag means it is omitted from the stored JSON entirely until the controller first writes it.

---

### 8.2 The Status Subresource

Having `Spec` and `Status` as two fields in the same Go struct does not by itself create any separation in the API. It is the `+kubebuilder:subresource:status` marker on the type that tells `controller-gen` to add this to the generated CRD:

```yaml
subresources:
  status: {}
```

Once that is present, the API server exposes two distinct endpoints for the resource:

```
/apis/apps.taskapp.io/v1alpha1/namespaces/{ns}/backends/{name}         ← main endpoint
/apis/apps.taskapp.io/v1alpha1/namespaces/{ns}/backends/{name}/status  ← status endpoint
```

The API server enforces the split at write time:
- A write to the main endpoint stores spec and metadata changes. Any status fields in the payload are **silently ignored**.
- A write to `/status` stores status changes. Any spec or metadata fields in the payload are **silently ignored**.

In the controller this maps to two different client calls:

```go
r.Update(ctx, backend)          // → main endpoint, updates spec
r.Status().Patch(ctx, backend)  // → /status endpoint, updates status
```

This is a **convention, not a hard Kubernetes requirement**. You can write a CRD without it and a single `r.Update()` will write both. The subresource exists to prevent a specific race condition: without it, a controller writing status with a stale copy of the object could silently overwrite a spec change that arrived between the controller's last `r.Get()` and its `r.Update()` call.

---

### 8.3 How `ReadyReplicas` Is Populated

The controller does not calculate readiness itself. It reads it from the Deployment that it owns:

```go
// updateStatus in backend_controller.go
deploy := &appsv1.Deployment{}
r.Get(ctx, types.NamespacedName{Name: deploymentName(backend), Namespace: backend.Namespace}, deploy)

backend.Status.ReadyReplicas = deploy.Status.ReadyReplicas
```

The value originates several layers down:

```
kubelet reports pod readiness to the API server
  → Deployment controller (inside kube-controller-manager) reads pod statuses
  → updates deploy.Status.ReadyReplicas
  → watch event fires → our operator reconciles
  → r.Get(deploy) reads deploy.Status.ReadyReplicas from the informer cache
  → copies that value into backend.Status.ReadyReplicas
  → r.Status().Patch(backend) writes it to the Backend's /status endpoint
```

The Backend operator is a relay. It promotes the Deployment's own status field up to the Backend level so that anyone watching a `Backend` CR gets a single object with the complete picture, without needing to also inspect the underlying Deployment.

---

### 8.4 How `QueueURL` Is Populated

The operator never calls AWS directly. The URL comes from Crossplane. When Crossplane provisions the SQS queue in AWS it writes the result back into the Queue CR's status:

```
Queue.status.atProvider.url = "https://sqs.eu-west-1.amazonaws.com/123456789/default-myapp-queue"
```

The `reconcileQueue` function reads this via the unstructured client:

```go
url, _, _ := unstructured.NestedString(existing.Object, "status", "atProvider", "url")
```

That URL is returned up through `reconcileSQS` into the main `Reconcile` function, which passes it to `updateStatus`:

```go
backend.Status.QueueURL = queueURL
```

The full chain:

```
Crossplane provisions SQS in AWS
  → writes Queue.status.atProvider.url
  → Queue watch fires → Backend reconcile triggered
  → reconcileSQS reads url from the Queue CR
  → updateStatus writes it to backend.Status.QueueURL
  → visible in kubectl get backend -o wide (QueueURL column)
```

---

### 8.5 Conditions

A single `ReadyReplicas` counter answers "how many pods are up?" but does not answer "is the queue ready?" or "is the deployment healthy vs still rolling out?". Conditions are the standard Kubernetes pattern for expressing multiple independent status dimensions on a single resource.

Each `metav1.Condition` has six fields:

| Field | Purpose |
|---|---|
| `Type` | Condition name — `"Available"`, `"SQSReady"` |
| `Status` | `"True"`, `"False"`, or `"Unknown"` |
| `Reason` | CamelCase machine-readable word — `"DeploymentAvailable"`, `"QueueProvisioning"` |
| `Message` | Human-readable string shown in `kubectl describe` |
| `LastTransitionTime` | Timestamp of the last **Status change** — not the last reconcile |
| `ObservedGeneration` | The `metadata.generation` this condition reflects |

The `LastTransitionTime` distinction matters: if `Available` has been `True` for an hour and a reconcile runs with nothing changed, the timestamp stays the same. It only advances when `Status` flips between `True`, `False`, and `Unknown`. This is what makes the field useful — it tells you how long the resource has been in its current state, not merely that the controller ran recently.

The `setCondition` helper in the controller implements an upsert: find the condition by `Type` and update it, or append a new one if not found. It also records a Kubernetes Event (visible in `kubectl describe`) only when the status actually transitions:

```go
func (r *BackendReconciler) setCondition(backend *appsv1alpha1.Backend, cond metav1.Condition) {
    existing := findCondition(backend.Status.Conditions, cond.Type)
    if existing != nil {
        if existing.Status != cond.Status {
            existing.LastTransitionTime = metav1.Now()  // only on transition
            r.Recorder.Event(...)                        // fire an Event
        }
        existing.Status = cond.Status
        existing.Reason = cond.Reason
        existing.Message = cond.Message
        existing.ObservedGeneration = cond.ObservedGeneration
    } else {
        backend.Status.Conditions = append(backend.Status.Conditions, cond)
    }
}
```

---

### 8.6 The `+listType=map` Markers on Conditions

```go
// +listType=map
// +listMapKey=type
Conditions []metav1.Condition `json:"conditions,omitempty"`
```

Without these markers, the conditions slice is treated as an ordered list for strategic merge patch purposes. Patching the list would replace it entirely, so updating `SQSReady` would also delete `Available`.

With `+listType=map` and `+listMapKey=type`, the API server treats the slice as a map keyed by the `type` field. Each condition is independently addressable by its type name. A patch that modifies only `SQSReady` leaves `Available` untouched. The generated CRD carries this instruction into the schema so the API server enforces it at the storage layer, not in your controller code.
