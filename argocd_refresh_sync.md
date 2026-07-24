# ArgoCD: Refresh vs Sync

## Core Loop

ArgoCD continuously reconciles two states:

```
Git (desired state)  ↔  Cluster (live state)
```

Every cycle: fetch Git → render manifests → compare → optionally apply.

---

## Refresh

**Refresh is a read-only operation.** It re-evaluates the application state without touching the cluster.

What it does:
- Fetches the latest Git revision
- Re-renders manifests (Helm, Kustomize, etc.)
- Computes the diff between desired and live state
- Updates the sync status shown in the UI (`Synced` / `OutOfSync`)

What it does **not** do:
- Apply any changes to the cluster
- Modify any resources

### Normal vs Hard Refresh

| | Normal | Hard |
|---|---|---|
| Git fetch | Uses cache if recent | Always re-fetches from remote |
| Manifest render | May use cached render | Forces re-render |
| Use when | Routine check | Debugging stale UI, cache issues |

### When to use refresh

- ArgoCD UI shows stale state and you need it to re-evaluate
- An external actor changed something (e.g. KEDA scaled a Deployment) and you want the diff to reflect reality
- Debugging a sync failure where the error message looks outdated

---

## Sync

**Sync is a write operation.** It drives the cluster toward the Git desired state.

What it does:
- Executes `kubectl apply` (or server-side apply)
- Creates, updates, and optionally deletes resources
- Enforces only the fields that are defined in Git — unspecified fields are left alone

### Sync Behaviours

| Option | Effect |
|---|---|
| `prune: true` | Deletes resources present in the cluster but no longer in Git |
| `selfHeal: true` | Re-applies if live state drifts from desired state |
| Manual sync | Triggered explicitly — no automatic re-application |
| Automated sync | Triggers automatically after a refresh detects `OutOfSync` |

### Important nuance — unspecified fields

If a field is **not declared in Git**, ArgoCD does not own it and will not revert it during sync. However, it may still show as `OutOfSync` because a diff exists.

Example: if `spec.replicas` is omitted from the Deployment manifest, ArgoCD syncs fine even if KEDA changes the replica count — it just won't touch that field.

---

## Refresh → Sync Flow

```
Refresh          Compare          Sync (optional)
   │                │                   │
Re-fetch Git  →  Compute diff  →  Apply to cluster
(read-only)     (read-only)        (write)
```

A refresh is always implied before a sync. Automated sync triggers automatically when a refresh returns `OutOfSync`.

---

## Annotations as Control Signals

ArgoCD is Kubernetes-native — instead of an external API, it reads `metadata.annotations` on `Application` resources as **imperative runtime signals**.

```
Annotation = one-time command to the ArgoCD controller
```

Key characteristics:
- Annotations are **not** part of Git desired state
- ArgoCD does **not** reconcile them (it won't revert them if you remove them)
- They are typically **removed automatically** by the controller after being processed
- They work with plain `kubectl` — no ArgoCD CLI or UI required

### Refresh Annotations

```bash
# Trigger a normal refresh
kubectl annotate application taskapp-frontend -n argocd \
  argocd.argoproj.io/refresh=normal --overwrite

# Trigger a hard refresh (bypass cache entirely)
kubectl annotate application taskapp-frontend -n argocd \
  argocd.argoproj.io/refresh=hard --overwrite
```

Both cause an immediate re-evaluation of the application state. No resources are modified.

---

## Mental Model

| Operation | Question it answers |
|---|---|
| **Refresh** | "What is the current drift between Git and cluster?" |
| **Sync** | "Make the cluster match Git." |
| **Annotation** | "Do this now, without changing Git." |

---

## Best Practices

- Use **refresh** when debugging stale state or after manual cluster changes
- Use **sync** to enforce desired state — never patch live objects directly if ArgoCD manages them
- Use **annotations** for one-off imperative actions (triggering a resync, forcing a refresh) — not for persistent configuration
- Keep ownership of fields explicit: if an autoscaler controls `spec.replicas`, remove it from the Git manifest so ArgoCD doesn't fight over it
