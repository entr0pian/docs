# Kubernetes Apply Models: Client-Side vs Server-Side Apply

## Overview

Kubernetes supports two apply models that determine how changes to objects are computed and merged.

| | Client-Side Apply | Server-Side Apply (SSA) |
|---|---|---|
| Merge strategy | 3-way diff (history-based) | Field ownership (declarative) |
| Tracking mechanism | `last-applied-configuration` annotation | `metadata.managedFields` |
| Size constraint | ~256KB annotation limit | None (API server handles it) |
| Multi-actor safety | Unsafe — actors overwrite each other | Safe — conflicts are explicit |
| Introduced | Original `kubectl apply` | Kubernetes 1.18+ |

---

## Client-Side Apply

### How it works

When you run `kubectl apply`, Kubernetes stores the entire submitted manifest as a JSON annotation on the object:

```
kubectl.kubernetes.io/last-applied-configuration: { ...full manifest... }
```

On the next apply, `kubectl` performs a **3-way merge** using three inputs:

1. **Last Applied** — the annotation (what you submitted previously)
2. **Live State** — the current object in the cluster
3. **New Manifest** — what you're applying now

The diff between New Manifest and Last Applied represents your *intent*, which is then applied on top of the live state.

### Example

```
Initial apply:   replicas: 3   → stored in annotation
Manual change:   replicas: 10  (via kubectl scale)
New apply:       replicas: 5

kubectl sees: 5 (new) vs 3 (last-applied) → user changed it
Result: replicas = 5  (manual change overwritten — intended)
```

### Problems

- **Annotation size limit (~256KB)** — large objects like CRDs can exceed it, causing the write to be rejected
- **Single-actor assumption** — only tracks `kubectl`; other managers (controllers, ArgoCD) are invisible to the diff
- **No field ownership** — cannot distinguish who is responsible for which field
- **Intent inference is fragile** — works by guessing what changed, not by knowing who owns what

---

## Server-Side Apply (SSA)

### How it works

The API server tracks **field ownership** rather than apply history. Each field in an object is tagged with the manager that owns it, stored in `metadata.managedFields`:

```yaml
metadata:
  managedFields:
    - manager: kubectl
      fields:
        f:spec:
          f:replicas: {}
    - manager: kube-controller-manager
      fields:
        f:status: {}
```

### Merge logic

For each field in the incoming manifest:

- **You own it** → update it
- **No owner** → you take ownership
- **Another manager owns it** → conflict (rejected unless `--force-conflicts` is passed)

No annotation is written. No history is needed.

### Example

```
Step 1: You apply spec.replicas = 3
        → you own spec.replicas

Step 2: Controller updates status.availableReplicas
        → controller owns status.*

Step 3: Another actor tries to set spec.replicas = 10
        → conflict: field already owned by you
```

To override: `kubectl apply --server-side --force-conflicts`

---

## Key Difference

| | Client-Side Apply | Server-Side Apply |
|---|---|---|
| Mental model | "What did the user change?" | "Who owns this field?" |
| Approach | Infer intent from history | Enforce ownership |
| Conflict handling | Silent overwrite | Explicit conflict error |

---

## Real-World Usage

**Who typically owns what:**

| Manager | Fields |
|---|---|
| User / GitOps tool | `spec.*` |
| Controllers | `status.*` |
| Mutating webhooks | Injected fields (sidecars, env vars) |

**ArgoCD:**
- By default does a 2-way diff (desired vs live), not a 3-way merge
- Can opt into SSA per application:

```yaml
syncOptions:
  - ServerSideApply=true
```

This is required for applications that ship large CRDs (e.g. `kube-prometheus-stack`, `keda`) — without it, ArgoCD's apply will be rejected due to the annotation size limit.

---

## ArgoCD: Two Separate "Server-Side" Settings

ArgoCD has two distinct server-side mechanisms that operate on different phases of the sync loop. They are easy to conflate but solve completely different problems.

### `ServerSideApply=true` — the write path

Set per application in `syncOptions`. Controls how ArgoCD **applies** resources to the cluster.

- ArgoCD sends the manifest to the API server, which merges it using field ownership (`managedFields`)
- No `last-applied-configuration` annotation is written
- Required for charts with large CRDs (`kube-prometheus-stack`, KEDA) that exceed the 256KB annotation limit

### `controller.diff.server.side: "true"` — the compare path

Set globally in `argocd-cmd-params-cm`. Controls how ArgoCD **computes diffs** before deciding whether a sync is needed.

Default (client-side) diff pipeline:

```
Fetch live resource from cluster
  → build typed value using ArgoCD's bundled static OpenAPI schema
  → compare with desired state
```

Server-side diff pipeline:

```
Send desired state to API server as dry-run ServerSideApply
  → API server returns what the object would look like after apply
  → ArgoCD diffs the response against live state
```

The API server always uses its own live schema, so ArgoCD's bundled schema is never consulted for field validation.

### Why having `ServerSideApply=true` is not enough

`ServerSideApply` only changes the **write path**. The comparison phase runs first — independently — and still uses ArgoCD's bundled static schema client-side. If that schema is out of date, the comparison fails before a sync is ever attempted.

Real example: Kubernetes 1.31 added `status.terminatingReplicas` to `ReplicaSet` (part of `DeploymentPodReplacementPolicy`, on by default from 1.32). ArgoCD's bundled schema didn't include this field. Every app that managed a `ReplicaSet` produced:

```
ComparisonError: error building typed value from live resource:
.status.terminatingReplicas: field not declared in schema
```

The app went `Unknown` — not `OutOfSync`, not `Synced` — because ArgoCD couldn't even reach the comparison step. `ServerSideApply=true` was already set on the affected apps (kube-prometheus-stack, KEDA) and had no effect on this error.

Enabling `controller.diff.server.side: true` fixed it by delegating schema authority to the API server, which knows every field in k8s 1.35.

### Summary

| Setting | Phase | Problem it solves |
|---|---|---|
| `ServerSideApply=true` (sync option, per app) | Write | CRD annotation size limit |
| `controller.diff.server.side: true` (global) | Compare | Stale bundled schema / unknown fields |

---

## One-Liners

- *"Kubernetes moved from a history-based diff model to an ownership-based conflict model."*
- *"3-way merge tries to infer intent; SSA enforces ownership."*
- *"`last-applied` stores what you said; `managedFields` tracks who owns what."*
- *"`ServerSideApply` changes how ArgoCD writes; `ServerSideDiff` changes how ArgoCD reads."*
