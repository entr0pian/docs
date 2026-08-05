# Cascading Deletion: Garbage Collection vs. Finalizers

Deleting a parent object and having Kubernetes clean up everything it manages is not one mechanism — it's two independent, non-overlapping ones, and which one applies to a given object is decided entirely by whether that object has anything in `metadata.finalizers`. Confusing the two is the easiest way to misdiagnose "why didn't this get deleted."

- **Owner references + the garbage collector (GC)** — the default, and the only one at play when `metadata.finalizers` is empty. A separate controller walks a graph built from `ownerReferences` and deletes dependents itself, with no involvement from the object's own controller.
- **Finalizers** — the opt-in mechanism. A non-empty `metadata.finalizers` list makes the API server defer the actual delete, giving a controller a window to run its own cleanup before the object is allowed to leave etcd.

The two worked examples below use one mechanism each, and the contrast between them — same-namespace Deployment cascade vs. cross-namespace ArgoCD `Application` cascade — is exactly what decides which one a given cleanup problem has to use.

---

## Mechanism 1: Owner References + The Garbage Collector

An object with no finalizers is deleted immediately: the API server purges it from etcd on the spot and emits exactly one watch event, `DELETED`. That event is delivered to *every* controller with an informer on that resource type, indiscriminately — watch delivery doesn't know or care which recipient has something useful to do with it. A resource's own controller (e.g. the Deployment controller, watching Deployments for its actual job of shifting replicas between ReplicaSets during a rollout) typically receives this event and does nothing with it: its reconcile runs, `Get()` returns `NotFound`, and it returns immediately. There is no cleanup branch to run, because the object is already gone.

The controller that actually does cleanup here is the **garbage collector**, running inside `kube-controller-manager`. It's a separate controller with its own informers spanning nearly every resource type in the cluster, and it maintains a continuously-updated, in-memory graph keyed by UID, built from every object's `ownerReferences`. That graph is not built at delete time — it's built (and kept current) the moment each dependent object is *created*: when a Deployment's controller creates a ReplicaSet, it stamps it with

```yaml
ownerReferences:
  - apiVersion: apps/v1
    kind: Deployment
    name: myapp
    uid: 3f2a9e10-...
    controller: true
    blockOwnerDeletion: true
```

and GC's own informer records that edge the moment it sees the `ADDED` event on the ReplicaSet — long before any deletion is in the picture. So when the owner's `DELETED` event eventually lands, GC isn't searching for dependents; it's a lookup against a graph node it already has.

Two constraints fall directly out of this design:

- **The cascade is a chain of hops, not one fan-out.** If a dependent's own dependents need to go too, GC has to walk each level separately, each time reacting to a `DELETED` event it generated on the previous hop.
- **`ownerReferences` carries no namespace field.** A namespaced object's owner is only ever resolved within that same namespace — there is no way to express "my owner lives elsewhere." This is the hard boundary that decides whether this mechanism can be used at all for a given cleanup problem; see the second worked example below for what happens when it can't.

`blockOwnerDeletion: true` is a related but separate knob: it only matters if the delete is issued with `propagationPolicy: Foreground`, in which case Kubernetes puts its own built-in `foregroundDeletion` finalizer on the *owner* itself, so the owner can't leave etcd until GC finishes the dependents. The default `kubectl delete` uses `Background` policy — the owner vanishes immediately and dependent cleanup trails behind it, exactly as traced below.

## Mechanism 2: Finalizers

A finalizer is a string in `metadata.finalizers`. Its presence changes what a delete request does: instead of purging the object, the API server sets `metadata.deletionTimestamp` and performs an **update**, leaving the object alive in etcd. That generates a `MODIFIED` event, not `DELETED` — the object still exists, so any controller watching it can still read it. A controller that recognizes its own finalizer name reacts to `deletionTimestamp != nil` by running whatever cleanup it owns, then removes its name from the list via its own `Update()` call. Once the finalizers list is empty *and* `deletionTimestamp` is set, the API server performs the real delete, and the final `DELETED` event fires.

`kubernetes_operators.md` §5 walks this exact sequence end to end with a hand-written operator as the worked example — a custom `Backend` CR carrying `apps.taskapp.io/finalizer`, added by the operator's own `Reconcile` function on first sight of the object and removed only after it has explicitly deleted the Deployment and Service it owns. The mechanics there are identical to what's described above; what the second worked example below adds is *why* a third-party controller needs this path in the first place, rather than just relying on `ownerReferences`.

---

## Worked Example: Same-Namespace Cascade — Deployment → ReplicaSet → Pod

Neither the Deployment, the ReplicaSet, nor the Pod carries a finalizer by default, so this is a pure Mechanism-1 cascade. A Deployment only owns its ReplicaSet directly — it has no `ownerReferences` relationship to the Pods at all — so GC has to walk two hops, not one:

```
kubectl delete deployment myapp        (no finalizers on Deployment/RS/Pod)
        │
        ▼
API server purges Deployment from etcd immediately — single DELETED event
        │
        ├──▶ Deployment controller's informer fires on the same event
        │       → Get(myapp) → NotFound → returns, nothing to do
        │
        ▼
GC's Deployment informer fires → graph lookup on myapp's UID → finds RS as
a dependent (edge recorded back when the RS was created, not now)
        │
        ▼
GC issues: DELETE ReplicaSet myapp-6f4c9b77d
        │
        ▼
RS purged from etcd — its own DELETED event
        │
        ├──▶ ReplicaSet controller's informer fires on the same event
        │       → Get(myapp-6f4c9b77d) → NotFound → returns, nothing to do
        │
        ▼
GC's ReplicaSet informer fires → graph lookup on the RS's UID → finds every
Pod it owns as dependents
        │
        ▼
GC issues: DELETE Pod myapp-6f4c9b77d-x2v9q  (one call per replica)
```

**The last hop isn't instant, even though nothing here has a finalizer.** A Pod delete carries a default grace period, and the API server's delete handling for Pods special-cases that: it sets `deletionTimestamp`/`Terminating` and keeps the object alive in etcd rather than purging it outright — the sequence `workloads.md`'s "Graceful Termination During Rollout" section covers in full (EndpointSlice removal, `preStop`, `SIGTERM`, then `SIGKILL` at the grace-period deadline or kubelet-confirmed exit). GC's job ends at issuing the `DELETE` call — the kubelet and the API server's own grace-period handling carry out the rest. It's a second, unrelated deferred-deletion mechanism sitting at the very end of the chain, superficially similar to a finalizer ("stay alive until something finishes") but built into the Pod resource's delete strategy specifically, not into `metadata.finalizers`.

## Worked Example: Cross-Namespace Cascade — ArgoCD Application → Managed Resources

`kubectl delete application taskapp-backend-prod` frequently surprises people: the `Application` object disappears, but the Deployment, Service, and everything else it managed keep running, untouched. That's the default, and it's a direct consequence of the namespace constraint above: an `Application` typically lives in the `argocd` namespace, and manages resources in an arbitrary destination namespace (`argocd.md` §3's `spec.destination.namespace`) — often several different ones across several Applications — and, in a multi-cluster setup, on an entirely different cluster (`spec.destination.server`) altogether. There is no `ownerReferences` entry that can express "my owner lives in a different namespace," so Mechanism 1 is structurally inapplicable here, independent of whether ArgoCD would even want automatic cascade as the default.

ArgoCD's answer is Mechanism 2, watched and acted on by the same `argocd-application-controller` that does normal sync reconciliation. The finalizer is `resources-finalizer.argocd.argoproj.io`, and — unlike the Deployment/RS/Pod case — it is **not present by default**:

```yaml
metadata:
  finalizers:
    - resources-finalizer.argocd.argoproj.io
```

It gets onto the object one of three ways: the ArgoCD UI's delete dialog has a "cascade" toggle, checked by default, which PATCHes this finalizer onto the `Application` before issuing the delete; `argocd app delete <name> --cascade` does the same from the CLI; or it's simply committed into the Application's manifest in Git, in which case every delete of it cascades unconditionally. A bare `kubectl delete application` against an object that never had the finalizer set skips all of this — same fast, no-finalizer path as the Deployment case, just with no GC and no `ownerReferences` chain to fall back on, so the managed resources are simply orphaned.

```
kubectl delete application taskapp-backend-prod
   (finalizers: [resources-finalizer.argocd.argoproj.io] already present)
        │
        ▼
API server: finalizers non-empty → sets deletionTimestamp, keeps the object
in etcd — an UPDATE, not a delete; no DELETED event yet
        │
        ▼
Application-CR informer (argocd.md §4) fires MODIFIED → app requeued
        │
        ▼
Reconcile sees deletionTimestamp != nil → branches into deletion handling
        │
        ▼
Walks the Application's resource tree (argocd.md §3) — built from the
tracking label/annotation for directly-applied resources, plus
ownerReferences for transitively-discovered ones, kept warm by the cluster
cache's watches (argocd.md §4), not re-listed now. This tree carries no
namespace restriction; it was never built from ownerReferences alone.
        │
        ▼
Issues DELETE against every resource in the tree, wherever it lives —
whatever namespace, even whatever destination cluster
        │
        ▼
Controller confirms, via the same cluster-cache watches, that every tracked
resource is actually gone — not just that the DELETE calls returned 200
        │
        ▼
Controller removes resources-finalizer.argocd.argoproj.io, r.Update()
        │
        ▼
API server: finalizers now empty + deletionTimestamp already set → purges
the Application from etcd → DELETED event fires
```

The confirm-before-finalizer-removal step matters: if the controller dropped the finalizer right after *issuing* the deletes, without checking they landed, a delete that silently failed (RBAC on the destination cluster, a target resource with its own blocking finalizer) would let the Application vanish while still orphaning a resource — the same end state as never having cascade enabled, just harder to notice because cascade was supposedly on.

**Two things this is not:**

- **Not `prune: true`.** That's an `argocd.md` §4 sync-time option — on every sync, it deletes whatever the tracking label/annotation says belongs to this Application but is no longer present in Git. It runs on ordinary syncs and has nothing to do with the `Application` object being deleted.
- **Not the same finalizer kubectl's own `--cascade=foreground` uses.** That flag puts Kubernetes' built-in `foregroundDeletion` finalizer on the *owner* object itself so *it* can't leave etcd until GC finishes the dependents — still a Mechanism-1, GC-driven path under the hood. ArgoCD's cascade delete also accepts a `--propagation-policy=foreground|background` flag with the same two names, but it's Argo's own finalizer and Argo's own controller loop doing the equivalent job by hand, not GC.

---

## What This Covers So Far

This covers the two independent mechanisms that answer "what happens to other objects when I delete this one": owner-reference-driven garbage collection (built continuously as objects are created, walked in hops, delivered via `DELETED` events that a resource's own controller receives but ignores) versus finalizer-gated controller cleanup (the delete request converted to an update, a `MODIFIED` event, explicit cleanup, then explicit finalizer removal). Two worked examples trace each one end to end — Deployment → ReplicaSet → Pod for the same-namespace, no-finalizer case, and an ArgoCD `Application` → its managed resources for the cross-namespace case that `ownerReferences` structurally cannot express, gated behind the opt-in `resources-finalizer.argocd.argoproj.io`.

Not yet covered: what happens when a tracked resource itself has a blocking finalizer that never clears (does the owning object sit `Terminating` forever, and is that surfaced anywhere short of `kubectl describe`); ArgoCD's `PreDelete`/`PostDelete` resource hooks and how they interleave with the deletion walk above; cascade behavior for an app-of-apps — does deleting the parent `Application` cascade through to child `Applications`' own managed resources, or just to the child `Application` objects themselves; the `orphan` propagation policy (neither example above uses it — it detaches dependents instead of deleting them, stripping their `ownerReferences` rather than issuing a `DELETE`); and Crossplane's use of finalizers for cloud-resource cleanup (`crossplane_architecture.md` mentions it in passing, but not as a full trace) as a third worked example of Mechanism 2 with its own twist — an external, non-Kubernetes resource (an AWS security group, say) that the finalizer's cleanup step has to reach out and delete via a cloud API, not just another Kubernetes object.
