# GitOps & ArgoCD

---

**Q: What is the GitOps model?**

GitOps is an operational model where Git is the single source of truth for both infrastructure and application configuration. The desired state of your system — Kubernetes manifests, Helm values, config — lives in Git. A controller runs inside the cluster and continuously reconciles the live state against what Git says should be running. If they diverge, the controller brings the cluster back in line.

**Before GitOps:**

A developer finishes a feature, CI builds the image, and the pipeline runs `kubectl apply` directly against the cluster. A few days later, someone hotfixes a ConfigMap manually in production. The pipeline never captures that change. Now Git no longer reflects reality — the cluster has drifted, and nobody knows when or why.

**After GitOps:**

The same developer opens a PR, gets it reviewed, and merges to main. The GitOps controller detects the new commit and applies it to the cluster. If someone manually edits a resource in the cluster, the controller detects the drift and reverts it within minutes. Git is always the authoritative record. Every change has an author, a timestamp, and a diff.

---

**Q: What is ArgoCD and how does it fit in the GitOps model?**

ArgoCD is a Kubernetes-native continuous delivery tool that implements the GitOps controller. It runs inside the cluster and continuously performs a reconciliation loop between the desired state stored in Git and the live state running in Kubernetes.

ArgoCD polls Git (or receives a webhook) and renders the manifests — whether plain YAML, Helm, or Kustomize. It then diffs the rendered output against what is actually deployed. If it detects drift, it marks the application as `OutOfSync`. Depending on the sync policy, it either waits for a manual sync or reconciles automatically using `autoSync`.

In the GitOps model, ArgoCD is the enforcement layer: Git expresses intent, ArgoCD makes the cluster reflect that intent, and it keeps doing so continuously.

---

**Q: Can you give a brief overview of the ArgoCD reconciliation loop?**

ArgoCD follows a reconciliation loop similar to native Kubernetes controllers, but with Git as the desired state source instead of etcd.

The loop is triggered by one of three things: a polling interval timeout, a Git webhook, or a manual refresh/sync.

**1. Read the Application CR**

The Application Controller reads the `Application` resource from Kubernetes to get the source (repo URL, revision, path), the destination (cluster, namespace), and the sync policy.

**2. Render manifests via the Repo Server**

The Application Controller makes a gRPC call to the Repo Server, which clones or fetches the Git repository, renders the manifests using Helm, Kustomize, or plain YAML, and returns the result as the desired state. For Helm this is equivalent to running `helm template` internally. The Repo Server caches both repositories and rendered output to avoid unnecessary work.

**3. Retrieve live state from Kubernetes**

The Application Controller fetches the current state of all managed resources from the Kubernetes API server.

**4. Normalized diff**

ArgoCD diffs desired state against live state — but not as a raw YAML comparison. It normalizes first by stripping fields like `resourceVersion`, `status`, `managedFields`, timestamps, and any server-defaulted values. It also applies any configured `ignoreDifferences` rules. This prevents false positives from fields Kubernetes mutates automatically.

Based on the diff, the application is marked either `Synced` or `OutOfSync`.

**5. Evaluate sync policy**

If `OutOfSync`, ArgoCD checks before acting:
- Is `autoSync` enabled?
- Is `selfHeal` enabled (required to revert manual cluster changes)?
- Is a sync window currently blocking deployment?
- What is the retry policy and current health state?

**6. Apply resources**

If sync is allowed, ArgoCD writes an `OperationState` to the Application CR and applies the manifests to Kubernetes. The apply strategy depends on sync options — `kubectl apply`, Server-Side Apply, replace, or delete/recreate.

**7. Evaluate health**

After applying, ArgoCD continuously checks the health of each resource: `Healthy`, `Progressing`, `Degraded`, or `Missing`. It updates the Application's sync status and health status accordingly.

The loop then repeats, ensuring the cluster continuously converges toward the desired Git state.

---

**Q: What is the OperationState in ArgoCD? Give a few examples.**

`OperationState` is a field written directly onto the `Application` CR by the Application Controller whenever a sync operation is in progress or has just completed. It acts as the live record of what ArgoCD is currently doing — or what it last did — to reconcile the application.

It captures:

- the operation type (sync)
- when it started and finished
- the overall phase (`Running`, `Succeeded`, `Failed`, `Error`)
- per-resource results — what was applied, what was skipped, what failed
- the sync strategy used (apply, replace, server-side apply)
- the retry state if the operation is being retried

**Example 1 — Successful sync**

A new commit is pushed. ArgoCD detects the application is `OutOfSync`, starts a sync, and writes `OperationState` with `phase: Running`. Once all resources are applied and healthy, it updates to `phase: Succeeded` with a timestamp and the list of resources that were synced.

**Example 2 — Failed PostSync hook**

A `PostSync` Job (e.g. a smoke test) exits with a non-zero code. The sync itself applied cleanly, but the hook failed. ArgoCD sets `OperationState.phase: Failed` and records the hook Job as the failing resource. The application stays `OutOfSync` until the issue is resolved.

**Example 3 — SelfHeal triggered**

Someone runs `kubectl edit deployment` and changes the replica count. ArgoCD detects drift during its next reconciliation cycle, writes a new `OperationState` with `initiatedBy: autoSync/selfHeal`, and reverts the replica count back to what Git specifies.

**Why it matters operationally**

`OperationState` is what the ArgoCD UI surfaces when you click into an application's sync status. It is also what the Notifications Controller reads to fire Slack or PagerDuty alerts on sync failures. In the CLI, `argocd app get <name>` shows the current operation state, making it the first place to look when a sync is stuck or failing.

---

**Q: How does the Application Controller track a running sync, and how can you stop it?**

When a sync starts — whether triggered by `autoSync`, a manual UI click, or the CLI — ArgoCD writes an `.operation` field directly onto the `Application` CR. This field is the authoritative signal that a sync is in progress. Everything else flows from it.

**How the controller tracks the operation**

The Application Controller runs an informer that watches all `Application` CRs in the `argocd` namespace. An informer is a Kubernetes-native mechanism: instead of polling the API server, it opens a long-lived watch stream and receives events (add, update, delete) as objects change. When the informer fires an event for an Application whose `.operation` field is newly set, the controller enqueues that application for sync processing.

The sync execution then writes live progress into `.status.operationState` on the same CR:

```yaml
status:
  operationState:
    phase: Running          # Running → Succeeded | Failed | Error | Terminating
    startedAt: "..."
    syncResult:
      resources:
        - name: taskapp-database-prod-seeder
          kind: Job
          status: Running
          hookPhase: Running
```

The controller continuously updates this field as resources are applied and hooks execute. The UI and CLI both read `.status.operationState` to display sync progress — they are not watching an internal process, just reading a Kubernetes object.

**What "stuck" looks like**

A PostSync hook Job (like the seeder job) blocks the sync in `Running` phase until it exits. If the Job fails repeatedly or is deleted externally while ArgoCD is watching it, the controller keeps waiting — `.status.operationState.phase` stays `Running` and `.operation` remains set on the CR. ArgoCD has no timeout for hook completion by default.

**How to stop it**

There are two approaches, both of which work by clearing or terminating the `.operation` field:

*Via the ArgoCD API (preferred):*
```bash
argocd app terminate-op <app-name>
```
This calls the ArgoCD API Server, which sets `.status.operationState.phase` to `Terminating` and signals the sync goroutine to stop. The controller detects the phase change, stops watching the hook, and clears `.operation` from the CR when it unwinds.

*Via direct Kubernetes patch (fallback — bypasses the API Server):*
```bash
kubectl patch app <app-name> -n argocd \
  --type merge -p '{"operation": null}'
```
This directly nulls `.operation` on the CR. The informer fires an update event, the Application Controller sees the field is gone, and stops the sync goroutine. The effect is immediate but bypasses the ArgoCD API Server's termination handshake — it is functionally equivalent for recovery purposes.

**What happens after termination**

Terminating a sync operation does not roll back any resources that were already applied — it only stops further progress. With `selfHeal: true` and `autoSync` enabled, the controller will re-evaluate the application on its next reconciliation cycle. If it finds the cluster is still `OutOfSync` (because a new commit arrived, or the hook failure left things incomplete), it will immediately queue a new sync.

This is the sequence we used when the seeder job was stuck on `taskapp-database-prod-secret not found`: deleting the Job cleared the running pod, but ArgoCD was still tracking the operation via `.operation` on the CR. Only after `terminate-op` nulled that field did the controller stop waiting and allow a fresh sync — which then ran the fixed seeder job template against the correct secret name.

---

**Q: How does ArgoCD differ from traditional CI/CD tools like Jenkins or Tekton?**

The fundamental difference is responsibility and direction. Jenkins and Tekton are **pipeline executors** — they run a sequence of steps in response to an event, and one of those steps typically pushes changes into the cluster. ArgoCD is a **continuous reconciler** — it has no pipeline, no steps, and nothing pushes. It pulls desired state from Git and continuously enforces it.

| | Jenkins / Tekton | ArgoCD |
|---|---|---|
| Model | Push-based pipeline | Pull-based reconciliation |
| Trigger | Event-driven (commit, PR, schedule) | Continuous loop + webhooks |
| Cluster access | CI system holds cluster credentials | Cluster pulls from Git — no inbound credentials |
| Drift handling | None — pipeline runs once and exits | Detects and corrects drift continuously |
| Rollback | Re-run old pipeline or manual apply | `git revert` — reconciler handles the rest |
| Scope | Build, test, package, deploy | Deploy and enforce only |
| State visibility | Pipeline logs | Live sync status, health, and diff in UI |

**They are complementary, not competing.** In a mature GitOps setup, Jenkins or Tekton handles the left side of the pipeline — building the image, running tests, updating the image tag in Git. ArgoCD handles the right side — detecting that Git changed and converging the cluster. The handoff point is a Git commit.

The key operational advantage of ArgoCD over a push-based tool is that the cluster is never in an unknown state. A Jenkins job that deploys and exits leaves no agent watching for drift. ArgoCD never stops watching.

---

**Q: If a manual change needs to happen in the prod cluster, how would you approach it?**

In a GitOps model, Git is the single source of truth, so ideally all production changes go through Git and are reconciled by ArgoCD. However, real incidents sometimes require immediate manual intervention — to mitigate an outage or stabilize a failing service before a proper fix can land.

**Prefer a Git-first fix whenever possible**

If the situation allows any lead time at all: commit the change, fast-track the PR review, and let ArgoCD reconcile normally. This keeps Git authoritative and leaves a full audit trail.

**If immediate intervention is required**

Temporarily prevent ArgoCD from reverting the manual change before it is applied. There are a few ways to do this depending on what level of control is needed.

*Disable automated sync entirely:*

```yaml
spec:
  syncPolicy: {}
```

*Or remove `selfHeal` and `prune` specifically:*

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

*Ignore a specific drifting field — common with HPA, KEDA, or emergency manual scaling:*

```yaml
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

*Block reconciliation for a set of applications using a Sync Window:*

```yaml
spec:
  syncWindows:
    - kind: deny
      schedule: "* * * * *"
      applications:
        - prod-*
```

*Trigger a manual refresh/recompare without syncing:*

```bash
kubectl annotate application myapp \
  argocd.argoproj.io/refresh=normal --overwrite
```

With reconciliation controlled, the temporary operational change can be applied safely.

**After stabilization**

1. Persist the final desired state back into Git.
2. Re-enable automated reconciliation.
3. Sync the application — the cluster converges back to Git as the single source of truth.

The key principle is that manual production changes are treated as controlled incident exceptions, not as a normal operational workflow. Every manual change should leave a paper trail and be followed by a Git commit that captures the outcome.

---

**Q: Can you explain ArgoCD's architecture components and how they interact with each other?**

ArgoCD follows a controller-based architecture where each component has a single, well-defined responsibility. The core design principle is Kubernetes-native: instead of a purpose-built API or database, ArgoCD uses Kubernetes custom resources as its data model and the Kubernetes API server as its coordination layer.

**Component overview**

| Component | Role |
|---|---|
| Application Controller | Core reconciliation loop — compares Git desired state vs cluster live state and drives convergence |
| Repo Server | Clones repositories, caches them, and renders manifests (Helm, Kustomize, Jsonnet, plain YAML) |
| API Server | Management plane — handles the UI, CLI, authentication, RBAC, and user-triggered operations |
| Redis | Cache layer — stores rendered manifests, application state, and session data to reduce repeated work |
| Dex *(optional)* | SSO bridge — connects ArgoCD to external identity providers (OIDC, LDAP, GitHub, SAML) |
| ApplicationSet Controller *(optional)* | Generates multiple `Application` resources from a single template — used for multi-cluster and environment fan-out |
| Notifications Controller *(optional)* | Sends alerts to Slack, PagerDuty, etc. on sync events, failures, or health changes |

**Application Controller**

This is the most important component. It implements the reconciliation loop by continuously watching `Application` CRs in Kubernetes. When a refresh is triggered (by polling, webhook, or annotation), it:

1. Calls the Repo Server via gRPC to get the rendered desired state
2. Queries the Kubernetes API for the current live state of all managed resources
3. Performs a normalized diff — ignoring `resourceVersion`, `status`, `managedFields`, timestamps, and server-defaulted fields to avoid false positives
4. Marks the application `Synced` or `OutOfSync`
5. If auto-sync or self-heal is enabled, applies the manifests back to the cluster

The Application Controller is, architecturally, just a native Kubernetes controller. The only difference from controllers like the Deployment controller is that **Git replaces etcd** as the desired state source.

**Repo Server**

The Application Controller does not render manifests itself. It delegates to the Repo Server via gRPC. The Repo Server:

- Clones or fetches the Git repository (with local caching to avoid redundant pulls)
- Renders manifests using the appropriate tool — for Helm this is conceptually equivalent to `helm template`, for Kustomize to `kustomize build`
- Caches both the repository and the rendered output keyed by revision + parameters
- Returns the rendered manifests to the Application Controller as the desired state

This separation keeps the controller stateless and makes manifest rendering independently scalable.

**API Server**

The ArgoCD UI and `argocd` CLI communicate with the API Server over REST/gRPC, not directly with the Application Controller. The API Server handles:

- User authentication (delegated to Dex for SSO)
- RBAC enforcement (who can sync, delete, override which applications)
- Translating user actions (press "Sync" in the UI) into operations on `Application` CRs, which the Application Controller then picks up
- Streaming application status back to the UI

**Redis**

Redis is the shared cache. It stores:

- Rendered manifest cache (so the Repo Server avoids re-rendering on every refresh)
- Application live state cache (so the UI loads quickly without hitting the Kubernetes API on every request)
- Session tokens for logged-in users

Without Redis, every page load in the ArgoCD UI would require a live Kubernetes API query and potentially a fresh `helm template` render.

**ApplicationSet Controller**

The ApplicationSet Controller enables the GitOps generator pattern. Instead of manually creating one `Application` CR per environment or cluster, you define a single `ApplicationSet` with a generator (e.g. list, Git directory, cluster) and the controller fans it out automatically. This is the standard approach for managing many applications across many clusters from a single App-of-Apps root.

**Interaction flow**

```
Git change (commit / webhook)
  │
  ▼
Application Controller detects refresh event
  │
  ├─── gRPC ──▶ Repo Server
  │                 │ clones / caches repo
  │                 │ renders manifests (helm template / kustomize build)
  │◀── rendered manifests ──┘
  │
  ├─── Kubernetes API ──▶ fetch live state of managed resources
  │
  ├─── normalized diff (ignore status, resourceVersion, managedFields)
  │
  ├─── mark Application: Synced / OutOfSync
  │
  └─── (if auto-sync) apply manifests to cluster via Kubernetes API
```

User-triggered actions (UI "Sync" button, `argocd app sync` CLI) pass through the API Server first, which writes an operation onto the `Application` CR. The Application Controller picks it up from there — the API Server never touches the cluster directly.

---

**Q: How do you manage multi-cluster or multi-environment setups in ArgoCD?**

The approach I've used is a dedicated management cluster running ArgoCD, combined with the App-of-Apps pattern where all child Application resources are generated from a central Helm chart.

**Management cluster**

ArgoCD lives in its own cluster with no application workloads. It holds credentials for every target cluster — dev, prod — and acts as the single control plane for all of them. Keeping ArgoCD separated means a workload cluster failure or compromise doesn't take down the GitOps control plane, and you have one place to audit all deployments across environments.

Each target cluster is registered via `argocd cluster add`, which creates a `ServiceAccount` with cluster-admin on the target and stores the token as a Secret in the management cluster's `argocd` namespace.

**App-of-Apps with a Helm chart**

Rather than managing Application resources manually per environment, I define all of them as Helm templates inside an `apps/` chart. Each template produces one ArgoCD `Application` CR. The entire chart is parameterized through a per-environment values file — things like `destinationServer`, secret paths, feature flags, and queue URLs.

The entry point is a single root manifest applied manually once per cluster:

```bash
kubectl apply -f root-prod.yaml
```

That root `Application` points ArgoCD at the `apps/` Helm chart with `values-prod.yaml`. ArgoCD renders it and creates all child Applications automatically — database, backend, frontend, monitoring, secrets operator — from that one apply.

**How multi-cluster routing works**

The `destinationServer` field in the values file is what routes each Application's resources to the correct cluster API server. The management cluster runs ArgoCD; everything else points at the target. Changing the values file is the only thing needed to retarget an entire environment.

One thing I learned the hard way: the ArgoCD Application name includes the environment suffix for identification in the UI (e.g. `taskapp-backend-prod`), but that name becomes the Helm `Release.Name` by default, which then leaks into every resource name — Services, Secrets, ConfigMaps. The fix is to set an explicit `releaseName` in each Application's Helm source so resource names stay env-agnostic regardless of what the Application is called.

**Sync wave ordering**

Within the apps chart I use sync waves to enforce bootstrap order across child Applications. Infrastructure operators like `kube-prometheus-stack` and `external-secrets` run in wave 0. Cluster-wide config like the `ClusterSecretStore` runs in wave 1 — it depends on the ESO operator already being ready. Application workloads run in wave 2. Without waves, ArgoCD would try to apply a `PrometheusRule` before the CRD exists, or a `ClusterSecretStore` before the operator is installed.

**Adding a new environment**

To onboard a new cluster I register it with `argocd cluster add`, create a new values file with the target's `destinationServer` and env-specific overrides, create a root manifest pointing at the apps chart with that values file, and apply it once. ArgoCD handles the rest — no app templates change, no manual Application creation.

---

**Q: How does `argocd cluster add` work under the hood, and what are the security implications?**

`argocd cluster add <context>` is the standard way to register an external cluster with a centralized ArgoCD instance. It is more than a config registration — it performs active operations on the target cluster and stores long-lived credentials in the management cluster.

**What it does, step by step**

1. Reads the named context from your local kubeconfig to get the target cluster's API server URL and CA certificate.
2. Connects to the target cluster and creates a `ServiceAccount` named `argocd-manager` in `kube-system`.
3. Creates a `ClusterRole` and `ClusterRoleBinding` granting `argocd-manager` **cluster-admin** across the entire cluster.
4. Creates a `Secret` of type `kubernetes.io/service-account-token` annotated with the SA — Kubernetes populates the `token` field automatically.
5. Reads that token and CA cert, then stores them as a `Secret` in the `argocd` namespace on the **management** cluster. From this point on, the management ArgoCD uses those credentials to communicate with the target cluster directly — no local kubeconfig needed.

**Why the token does not expire**

There are two distinct token types in Kubernetes:

| Type | How created | Expiry |
|---|---|---|
| Bound token | `kubectl create token` / projected volume | Short-lived (default 1 hour), audience-scoped |
| SA token Secret | `type: kubernetes.io/service-account-token` | **Never expires** |

`argocd cluster add` deliberately creates the second type — an explicit SA token Secret. Because it is not a bound token, Kubernetes never rotates or expires it. The token remains valid for the lifetime of the Secret and ServiceAccount, regardless of how much time passes.

This is intentional: ArgoCD needs a stable, always-valid credential it can use at any time without a refresh mechanism. The trade-off is that the token is permanent.

**Security dangers**

*Cluster-admin is too broad.*
The ClusterRoleBinding grants `argocd-manager` full cluster-admin on the target cluster — the same level as `system:masters`. ArgoCD only needs to read, create, update, and delete application-managed resources in specific namespaces. Cluster-admin gives it the ability to do anything to any resource in any namespace, including reading all Secrets, modifying RBAC, and deleting system resources. This violates the principle of least privilege.

*A stolen token is a permanent key.*
Because the token never expires, if an attacker exfiltrates it — from a backup, a misconfigured Secret, or a compromised ArgoCD instance — they have permanent cluster-admin access to the target cluster until someone manually deletes the Secret or SA. There is no automatic expiry or rotation to limit the blast radius.

*The management cluster becomes a high-value target.*
ArgoCD stores the credentials for every registered cluster as Secrets in the `argocd` namespace. If the management cluster is compromised, an attacker gains cluster-admin credentials for all registered clusters simultaneously. The management cluster's security posture directly determines the blast radius of a breach.

*No built-in rotation.*
ArgoCD does not rotate cluster credentials. Once registered, the same token is used indefinitely. Manual rotation requires removing and re-adding the cluster.

---

**Q: Why did custom resources show `Synced` with no Health status at all, and how do you give ArgoCD a way to evaluate them?**

After a full cluster reset, `argocd app get taskapp-backend-prod` showed the `Backend` custom resource (and, nested under it in the resource tree, the `RDSInstance` and `SQSQueue` Crossplane claims it owns) as `Synced` — but the Health column was completely blank. Not `Missing`, not `Unknown`, just empty.

**Why "blank" and not some other state**

ArgoCD's health evaluation (step 7 of the reconciliation loop, above) only produces a value for resource kinds it knows how to assess. That knowledge comes from two places:

1. **Built-in Go checks** — hardcoded for well-known kinds: `Deployment`, `StatefulSet`, `Ingress`, `Job`, `PVC`, `HorizontalPodAutoscaler`, and a handful of others.
2. **Lua scripts registered in `argocd-cm`** — under `resource.customizations.health.<group>_<kind>`, opt-in per exact group/kind.

For any kind that falls into neither bucket — which includes almost every custom CRD, including ones from popular tools like cert-manager or Crossplane — ArgoCD has no generic fallback. It doesn't inspect `.status.conditions` on its own, even though that's a widely-used Kubernetes convention (it's what `kubectl wait --for=condition=Ready` and Flux's kstatus-based health checks rely on generically, without per-CRD configuration). ArgoCD's design deliberately requires an explicit opt-in instead, so the Health field is simply never written — hence blank, not a specific "unknown" state.

**The complication: no single condition-name convention**

The natural first instinct is "write a Lua script that checks for a `Ready` condition." That doesn't generalize here, because our own operators don't agree on a name:

| Resource | Group | Conditions used |
|---|---|---|
| `Backend` | `apps.taskapp.io` | `RDSReady`, `SchemaReady`, `Available`, `SQSReady` (see [`kubernetes_operators.md` §8.5](./kubernetes_operators.md)) |
| `RDSInstance` / `SQSQueue` | `database.taskapp.io` / `queue.taskapp.io` | `Synced`, `Ready` (Crossplane's own convention) |

`Backend`'s `Available` condition specifically only reflects `Deployment.Status.ReadyReplicas >= desired` — it is not an aggregate of the other three conditions. So neither "look for `Ready`" nor "look for `Available`" is a rule that covers all three kinds.

**The fix: condition-agnostic health, not name-specific health**

Since every condition here is a standard `metav1.Condition` (`Type`, `Status`, `Reason`, `Message`), the generalizable rule doesn't need to know the condition names at all — it just needs to check that *whatever* conditions are present are all `True`:

- any condition `status: "False"` → `Degraded`
- `status.conditions` missing/empty, or some condition neither `True` nor `False` (e.g. `Unknown`) → `Progressing`
- all present conditions `True` → `Healthy`

This logic was added once in `bootstrap-cluster/install-argocd/values.yaml`, under `argo-cd.configs.cm`, and reused for all three group/kinds via a YAML anchor/alias (resolved by the YAML parser before Helm ever sees the values — not a Helm templating trick):

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

Applied with the same idempotent command used to install ArgoCD (`helm upgrade --install argocd bootstrap-cluster/install-argocd --namespace argocd --kube-context kind-management`). After a hard refresh (`argocd app get taskapp-backend-prod --hard-refresh`), all three resources report `Healthy`, matching their live `.status.conditions`.

**Why it matters**

`Synced` only means ArgoCD's last apply matched Git — it says nothing about whether the RDS instance actually provisioned or the schema migration succeeded. Health is the only signal that would have caught a resource stuck mid-provisioning, and it also gates sync-wave ordering (later waves should wait on earlier ones being healthy, not just synced).

This also sets the pattern for every future operator: the requirement isn't "name your condition `Ready`," it's "use `metav1.Condition` for status." Health then follows automatically by adding one more aliased key pointing at the same shared script — no new Lua to write per CRD.