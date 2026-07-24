# KEDA Admission Webhook — Stale Cache After HPA Deletion

## The Error

After migrating the frontend from a native `HorizontalPodAutoscaler` to a KEDA `ScaledObject`, ArgoCD failed to sync:

```
admission webhook "vscaledobject.kb.io" denied the request:
the workload 'taskapp-frontend' of type 'apps/v1.Deployment'
is already managed by the hpa 'taskapp-frontend-hpa'
```

---

## Root Cause

KEDA's admission webhook validates every `ScaledObject` on creation. One of its checks is: *"is the target Deployment already controlled by an HPA that KEDA doesn't own?"*

The old `taskapp-frontend-hpa` (created directly by the previous Helm chart) was still present in the cluster. ArgoCD's `prune: true` would eventually remove it, but apply runs before prune — so the webhook saw the old HPA and rejected the `ScaledObject`.

### Fix — Step 1: Delete the old HPA manually

```bash
kubectl delete hpa taskapp-frontend-hpa -n default
```

---

## The Second Problem — Stale Webhook Cache

After deleting the HPA and triggering a resync, the webhook **still** rejected with the same error. The HPA was confirmed gone from the cluster:

```bash
kubectl get hpa -A   # No resources found
```

But the webhook logs still reported:

```
validation error: the workload 'taskapp-frontend' is already managed by the hpa 'taskapp-frontend-hpa'
```

**Root cause:** KEDA's admission webhook is built on [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime), which uses an in-memory informer cache to avoid hitting the API server on every admission request. This cache is populated at startup via a full list, then kept in sync via watch events. The running pod still held the deleted HPA in its cache — the delete event had not been processed before ArgoCD retried the sync.

### Fix — Step 2: Restart the webhook deployment

```bash
kubectl rollout restart deployment/keda-admission-webhooks -n keda
```

On startup, controller-runtime performs a fresh list-and-watch sync against the API server before the webhook begins serving traffic. The new pod's cache initialised correctly with no HPAs, and the next sync succeeded.

---

## Key Takeaways

- When migrating away from a manually-managed HPA, delete it explicitly before applying the `ScaledObject` — ArgoCD prune ordering cannot be relied on to do it first
- If a KEDA webhook rejects based on a resource that no longer exists in the cluster, restart the webhook pod to force a fresh cache initialisation
