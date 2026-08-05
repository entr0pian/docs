# ArgoCD

This is the canonical ArgoCD doc — new ArgoCD material goes here going forward, rather than spawning a new file per topic (see `gitops_and_argocd.md` and `argocd_refresh_sync.md` for earlier, narrower entries written before this one existed).

---

## 1. What ArgoCD Is

ArgoCD is a Kubernetes-native GitOps controller: it continuously reconciles the desired state declared in a Git repository against the live state of a Kubernetes cluster, and converges the cluster toward Git whenever they diverge.

The core mental model: the **Application Controller** — the component that actually does this reconciliation — is architecturally just a native Kubernetes controller, built the same way as the Deployment controller or ReplicaSet controller. It watches a custom resource (`Application`), compares desired vs. live state, and acts on the difference. The only thing that's different from a stock controller is *where the desired state comes from*: a normal controller reads the object's own `spec` from etcd; ArgoCD instead treats a Git repository (a specific revision, path, and rendering tool) as the desired-state source, and etcd only holds the live/actual side of the comparison.

This makes ArgoCD **pull-based**, which is the key operational distinction from push-based CI/CD (Jenkins, Tekton, a pipeline step that runs `kubectl apply`):

| | Push-based pipeline | ArgoCD (pull-based reconciler) |
|---|---|---|
| Who initiates the change | The CI system, holding cluster credentials | The cluster-side controller, pulling from Git |
| What happens after deploy | Pipeline exits — nobody is watching | Controller keeps watching, forever |
| Drift (manual `kubectl edit`) | Invisible — Git and cluster silently diverge | Detected next reconcile, optionally auto-reverted (`selfHeal`) |
| Credentials | CI needs write access to the cluster | Cluster needs read access to Git — no inbound deploy credentials |

Nothing pushes into the cluster from outside; the controller inside the cluster pulls, decides, and applies. That's the whole model — everything else in this doc is what actually implements it.

---

## 2. What a Standard ArgoCD Helm Install Deploys

Installing the common `argo-cd` Helm chart doesn't deploy "an ArgoCD" — it deploys several independent workloads, each with a narrow responsibility, coordinating only through Kubernetes objects (CRDs, ConfigMaps, Secrets) rather than a private database.

| Component | Workload kind | Role |
|---|---|---|
| `argocd-application-controller` | **StatefulSet** | The reconciliation loop itself — watches `Application` CRs, diffs desired vs. live, drives sync. Stateful because of cluster sharding (below). |
| `argocd-repo-server` | Deployment | Clones/fetches Git repos, renders manifests (Helm/Kustomize/Jsonnet/plain YAML), caches both. Stateless, horizontally scalable. |
| `argocd-server` (API Server) | Deployment | Management plane for the UI/CLI — auth, RBAC, translates user actions into writes on `Application` CRs. Never touches the target cluster directly. |
| `argocd-redis` | Deployment (or Redis+Sentinel in HA mode) | Shared cache — rendered manifests, live-state snapshots, UI session tokens. |
| `argocd-dex-server` | Deployment *(optional)* | SSO bridge to external identity providers (OIDC/SAML/LDAP/GitHub). Only deployed if SSO is configured. |
| `argocd-applicationset-controller` | Deployment *(optional, commonly enabled)* | Generates many `Application` resources from one `ApplicationSet` template — the multi-cluster/multi-env fan-out mechanism. |
| `argocd-notifications-controller` | Deployment *(optional)* | Watches Application status changes and fires Slack/PagerDuty/etc. alerts. |

**CRDs installed:**
- `Application` — one instance per deployable unit (source repo/path/revision + destination cluster/namespace + sync policy).
- `AppProject` — a tenancy boundary; restricts which repos, clusters, and resource kinds a group of Applications may use.
- `ApplicationSet` — the Application-generator template, if the ApplicationSet controller is enabled.

**Key ConfigMaps/Secrets (this is where ArgoCD's "no private database" claim is literal — all runtime config lives in plain Kubernetes objects):**
- `argocd-cm` — main config: repository credentials, resource health customizations (Lua scripts), the resource-tracking method (label vs. annotation, see §3).
- `argocd-rbac-cm` — RBAC policy (who can sync/delete/override which Applications).
- `argocd-secret` — admin credential hash, TLS certs, Dex client secrets.
- `argocd-cmd-params-cm` — startup flags for the other components (e.g. controller sharding algorithm, repo-server parallelism).

**Why the Application Controller is a StatefulSet and not a Deployment:** when scaled beyond one replica, ArgoCD shards work by *target cluster*, not by Application — each controller replica owns a fixed subset of the registered clusters, assigned by hashing the cluster's API server URL against a stable replica identity. A Deployment's pods have no stable identity to hash against; a StatefulSet's ordinal does.

---

## 3. The Application Resource Tree — How Managed Resources Are Discovered

Before the reconcile flow in §4 makes sense, one thing has to be pinned down: the Application Controller has no built-in notion of "everything under this Helm chart belongs to app X." Every resource in an Application's tree gets there through one of exactly two mechanisms.

**Directly tracked resources** — anything ArgoCD itself applies gets stamped with a tracking marker:

- **Legacy/default:** the label `app.kubernetes.io/instance: <app-name>`.
- **Recommended today:** the annotation `argocd.argoproj.io/tracking-id`, which encodes `<app-name>:<group>/<kind>:<namespace>/<name>`. Moving to an annotation avoids colliding with a user's own `app.kubernetes.io/instance` label (a very common label to already be using for something else), and it carries more structure than a bare label value.

**Transitively discovered resources** — anything *not* directly applied by ArgoCD (a Deployment's ReplicaSet, a custom operator's child objects it creates during its own reconcile) carries no tracking marker at all. These are linked into the tree purely by walking `ownerReferences` back to something that *is* tracked. This is also the mechanism the cluster cache in §4 relies on to attribute a watched resource's change back to the right `Application`.

### Gotchas

- **A missing `ownerReference` doesn't just cost you health — it costs you everything.** If an operator creates a child object without setting `ownerReferences` back to its parent CR, that child is invisible to the tree: not shown in the Application's resource view, not included in the normalized diff, not counted in health aggregation (§5), and not eligible for `prune: true` cleanup. Setting `ownerReferences` on every child object is a correctness requirement for any custom resource you want ArgoCD to manage cleanly — not a nice-to-have.
- **Being discovered and having a health verdict are two separate things.** A resource can be fully present in the tree (via either mechanism above) and still contribute nothing to the Application's aggregate health, if its Kind has no registered health check — see §5. This is exactly what happened in the CRD-health incident documented in `gitops_and_argocd.md`: `Backend`, `RDSInstance`, and `SQSQueue` were all correctly discovered and diffed, but showed a blank Health column because no check existed yet for those kinds.
- **Discovery is what makes `Synced` meaningful; it's independent of what makes `Healthy` meaningful.** The normalized diff in §4 runs against every discovered resource, tracked or transitive, regardless of whether that resource's kind has a health check registered. Sync status and health status are computed from the same tree but are otherwise unrelated signals.

This same tracking mechanism — not `ownerReferences` alone, since those can't cross a namespace boundary and an `Application`'s managed resources routinely live outside its own namespace — is also what makes deleting an `Application` able to cascade to everything it manages, gated behind the opt-in `resources-finalizer.argocd.argoproj.io` finalizer. `cascading_deletion.md` has the full trace, contrasted against the same-namespace, no-finalizer, `ownerReferences`-only case (a Deployment cascading to its ReplicaSet and Pods).

---

## 4. Deep Dive: The Self-Heal / Drift-Detection Flow

Scenario: an `Application` is `Synced`. Someone runs `kubectl edit deployment` directly against a Deployment it manages, changing something Git doesn't say. Here's the full mechanical path from that edit back to the cluster being reverted.

### There are two separate watches, not one

It's tempting to assume the Application Controller only watches `Application` CRs (since that's the object type its reconcile loop is nominally "about"), and that it discovers drift by reaching out and polling child resources when it feels like it. That's not how it's wired:

1. **Application-CR informer** — watches `Application` objects in the `argocd` namespace. This is what tells the controller "a sync was requested" or "the spec changed."
2. **Cluster cache** — a *separate* set of dynamic informers, opened per registered target cluster, watching (almost) every API resource type in that cluster: Deployments, ReplicaSets, Pods, Services, CRDs, all of it (subject to configured include/exclude rules). This builds a live, in-memory resource tree, kept current purely by watch events — not by re-polling.

The cluster cache is what makes the rest of this flow possible: by the time a diff needs to happen, the live state is already sitting in memory, kept fresh by watches, indexed by the tracking label/annotation from §3.

### Step by step

```
kubectl edit deployment (manual drift)
        │
        ▼
Cluster-cache watch on Deployments fires (already-open informer, no new watch created)
        │
        ▼
Cache entry for that Deployment is invalidated/updated
        │
        ▼
Controller resolves owning Application via tracking-id → requeues that Application
        │
        ▼
Application Controller reconcile runs:
   ├─ gRPC → Repo Server: fetch Git revision, render manifests (helm template / kustomize build)
   ├─ read live state — from the cluster cache (already current), not a fresh live GET
   ├─ normalize both sides (strip resourceVersion, status, managedFields, timestamps)
   └─ diff
        │
        ▼
Diff is real → write .status.sync.status = OutOfSync onto the Application CR
        │
        ▼
Sync policy check: autoSync + selfHeal enabled?
        │
        ▼  yes
Apply the rendered manifest for the drifted resource — a full apply (kubectl-apply / SSA
semantics: three-way merge against the live object), not a patch and not a replace
        │
        ▼
Deployment reverted to Git's declared state; health re-evaluated afterward
```

### Correcting the two easiest misconceptions here

- **`OutOfSync` is not a label on the Deployment.** It's a status field (`.status.sync.status`) on the *Application* object. The Deployment itself carries no ArgoCD-specific sync-state marker — only the tracking label/annotation from §3.
- **The controller isn't "watching for its own status write."** The diff is computed and the `OutOfSync` status is written in the same reconcile call — there's no round-trip where the controller notices a status change on the child resource and loops back in. The watch-driven trigger in this flow is on the *managed resource being externally edited*, not on ArgoCD's own status output.

### Why "full apply, not a patch" matters

Sync sends the complete rendered manifest and applies it the way `kubectl apply` does — a three-way merge (using the last-applied-config annotation, or Server-Side Apply field ownership if `ServerSideApply=true` is set) — rather than a raw JSON Patch or a full `replace`. That merge semantics is exactly why a field genuinely absent from Git (e.g. `spec.replicas` when an HPA/KEDA owns it) survives a sync untouched: the apply only asserts the fields present in the rendered manifest, it doesn't overwrite the object wholesale. An opt-in `Replace=true` sync option exists for a full delete/recreate, but it isn't the default path.

---

## 5. Application Health

Health is a separate signal from sync status: `Synced` only means the last apply matched Git — it says nothing about whether the resources that were applied are actually working. Health is what answers that.

### 5.1 The Status Values

| Status | Meaning | Typical trigger |
|---|---|---|
| `Healthy` | Live state matches the desired end-state | Deployment: ready/available/updated replicas all match desired, no stalled condition |
| `Progressing` | Actively moving toward desired state, nothing wrong yet | Rollout in progress; new Pods not yet ready; a custom resource's `status.conditions` not yet populated |
| `Degraded` | Explicitly failed or stuck | A Deployment's `Progressing` condition reporting `ProgressDeadlineExceeded`; a Job that failed; a custom resource with a condition explicitly `status: "False"` |
| `Suspended` | Intentionally paused, not broken | `Job.spec.suspend: true`; a paused Argo Rollout waiting for manual promotion |
| `Missing` | Declared in Git, not present live at all | Resource hasn't been created yet, or was deleted externally |
| `Unknown` | ArgoCD tried to assess and couldn't | A Lua health script throws an error during evaluation |

A seventh case sits outside this table entirely: a resource whose Kind has **no health check registered at all** (built-in or Lua) gets no value here — not `Unknown`, just blank, and it doesn't participate in the aggregation below either way (see the CRD-health gotcha in §3).

### 5.2 How ArgoCD Reasons About a Single Resource's Health

There's no generic convention ArgoCD reads off every object — no universal "status: OK" field. Every Kind's health comes from one of two sources:

**Built-in Go checks**, for a fixed set of well-known kinds ArgoCD ships hardcoded logic for (Deployment, ReplicaSet, StatefulSet, DaemonSet, Pod, Job, PVC, Service, Ingress, and others). For Deployment specifically, this is roughly: if `status.observedGeneration` lags `metadata.generation`, the controller hasn't even seen the latest spec yet → `Progressing`; if the `Progressing` condition reports `status: False` (reason `ProgressDeadlineExceeded`) → `Degraded`; if `updatedReplicas`/`availableReplicas` haven't caught up to `spec.replicas` yet → `Progressing`; otherwise → `Healthy`.

**Lua scripts**, for everything else — in practice, every CRD. Registered in `argocd-cm` under a key scoped to the exact API group and kind:

```
resource.customizations.health.<group>_<kind>: |
  <lua>
```

The script receives an implicit global, `obj` — the live resource, structured as a Lua table mirroring the exact JSON the API server returns (`obj.status.conditions`, `obj.spec.replicas`, whatever the CRD's schema has). It must return a table with a `status` field set to one of the values in §5.1, and optionally a `message` string surfaced in the UI/CLI. If the script errors or never sets `status`, that resource's health becomes `Unknown`.

The real example already in this repo, from `gitops_and_argocd.md` — a single condition-agnostic script reused across three custom resources via a YAML anchor:

```yaml
resource.customizations.health.apps.taskapp.io_Backend: &conditionsHealthCheck |
  local hs = {}
  if obj.status == nil or obj.status.conditions == nil or #obj.status.conditions == 0 then
    hs.status = "Progressing"
    hs.message = "Waiting for status.conditions"
    return hs
  end
  local degraded = {}
  local progressing = false
  for i, condition in ipairs(obj.status.conditions) do
    if condition.status == "False" then
      table.insert(degraded, (condition.type or "condition") .. ": " .. (condition.message or ""))
    elseif condition.status ~= "True" then
      progressing = true
    end
  end
  if #degraded > 0 then
    hs.status = "Degraded"
    hs.message = table.concat(degraded, "; ")
  elseif progressing then
    hs.status = "Progressing"
  else
    hs.status = "Healthy"
  end
  return hs
resource.customizations.health.database.taskapp.io_RDSInstance: *conditionsHealthCheck
resource.customizations.health.queue.taskapp.io_SQSQueue: *conditionsHealthCheck
```

It doesn't look for a specific condition name (`Backend`, `RDSInstance`, and `SQSQueue` don't even agree on one) — it just checks that whatever `metav1.Condition`s are present are all `True`. That's what makes one script reusable across three unrelated CRDs via the alias.

### 5.3 Aggregation — Worst Status Wins Across the Tree

An Application's overall health is the worst status found among every resource in its tree that actually produced a verdict (§3's "discovered but unassessed" resources are excluded from this comparison entirely, not counted as passing). Concretely: if anything in the tree reports `Degraded`, the Application reports `Degraded`, regardless of how many other resources say `Healthy`. If nothing is `Degraded` but something is still `Progressing`, the Application reports `Progressing`. Only when every resource that produces a verdict says `Healthy` does the Application read `Healthy`.

### 5.4 Example — A Built-in Check Bubbling `Degraded` Through the Tree

Using the `taskapp-backend-prod` tree from §3: `Backend`, `RDSInstance`, and `SQSQueue` are all fully provisioned and their Lua checks all report `Healthy`. The Deployment has 2 desired replicas; one Pod is healthy, the other is stuck in `CrashLoopBackOff`.

```
Backend        (Lua)        → Healthy
├─ RDSInstance (Lua)        → Healthy
├─ SQSQueue    (Lua)        → Healthy
└─ Deployment  (built-in)   → Progressing   (availableReplicas < spec.replicas)
   └─ ReplicaSet             (no check registered — excluded from aggregation)
      ├─ Pod #1 (built-in)  → Healthy
      └─ Pod #2 (built-in)  → Degraded      (CrashLoopBackOff, Ready: False)

Application aggregate → Degraded
```

The worst verdict in the tree came from a built-in check three levels down, not from any of the Lua-derived ones sitting at the top — and the aggregation doesn't distinguish where a verdict came from. A `Degraded` Pod overrides three `Healthy` Lua results just as readily as a `Degraded` custom resource would override three healthy Pods.
