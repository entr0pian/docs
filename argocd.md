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

## 3. Resource Tracking — How an `Application` Connects to Its Deployment

Before the reconcile flow in §4 makes sense, one thing has to be pinned down: the Application Controller has no built-in notion of "everything under this Helm chart belongs to app X." It has to be told, and it's told via a marker stamped onto every resource it applies.

- **Legacy/default:** the label `app.kubernetes.io/instance: <app-name>`.
- **Recommended today:** the annotation `argocd.argoproj.io/tracking-id`, which encodes `<app-name>:<group>/<kind>:<namespace>/<name>`. Moving to an annotation avoids colliding with a user's own `app.kubernetes.io/instance` label (a very common label to already be using for something else), and it carries more structure than a bare label value.

This marker is the index key for everything in §4 — it's how a resource discovered via a cluster-wide watch gets attributed back to a specific `Application`.

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
