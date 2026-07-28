# Workload Controllers: Deployment, StatefulSet, and DaemonSet

Deployment and StatefulSet both manage a set of Pods from a template and
reconcile actual replica count toward a desired number you set. The
difference between them is entirely about **identity**: whether an
individual Pod's storage and network address need to survive across
restarts and be distinguishable from its siblings, or whether every Pod is
fully interchangeable.

DaemonSet doesn't sit on that same spectrum — it answers a different
question entirely (placement, not identity or interchangeability), covered
in its own section below.

## Deployment: Fungible Pods, No Identity

A Deployment owns a ReplicaSet, which owns Pods created with a random name
suffix (`api-7d9f8c6b45-x2v9q`). No Pod has a durable identity — if it's
deleted and recreated, the replacement gets a new name, and (unless the Pod
spec references a pre-existing PVC by name, which all replicas would then
have to share) a fresh, empty volume if one is even defined.

**Pod selection during scale-down/rollout is not random.** The ReplicaSet
controller ranks its Pods and deletes off one end of that ranking. The
ranking prioritizes, roughly in order:

1. Not-yet-scheduled or not-Running Pods before Running ones.
2. Not-Ready Pods before Ready ones — an unhealthy Pod is the first thing
   killed.
3. A lower `controller.kubernetes.io/pod-deletion-cost` annotation value
   before a higher one — the explicit manual override, if set.
4. Pods on a node holding more replicas of this ReplicaSet before pods on a
   node holding fewer — keeps the replica spread across nodes even as it
   shrinks.
5. Among remaining ties, a Pod that's been Ready for less time before one
   Ready longer — a freshly started Pod is judged less proven than a
   long-running one and goes first.

So "no identity" doesn't mean "no logic" — there's a deterministic priority
function, it just isn't keyed on which specific replica a Pod is. Any
replica can serve any request, so the controller optimizes for health and
balance instead of preserving a particular instance.

### Update Flow

A Deployment orchestrates ReplicaSets. Every change to a Deployment's
`PodTemplateSpec` produces a new ReplicaSet — the old one is kept at 0
replicas as a rollback revision. Kubernetes never mutates a ReplicaSet
in-place; it always creates a new one.

> **Key Idea:** Every Deployment revision = a distinct ReplicaSet. Rollback
> = re-scaling an old ReplicaSet back up.

**1. Initial State**

```
Deployment: myapp
ReplicaSet:  myapp-7d6f5d8b7   →   3 × nginx:1.0
```

**2. User Triggers Update**

```bash
# e.g. image change: nginx:1.0 → nginx:2.0
kubectl apply -f deployment.yaml
```

**3. Deployment Controller Detects the Change**

`PodTemplateSpec` has changed. Because ReplicaSets are treated as
**immutable rollout revisions**, the controller does **not** modify the
existing ReplicaSet — it creates a new one.

```
Old RS:  myapp-7d6f5d8b7   (nginx:1.0)
New RS:  myapp-6f4c9b77d   (nginx:2.0)  ← created, starts at 0 pods
```

**4. Rolling Update**

Example strategy: `maxSurge: 1`, `maxUnavailable: 1`, `desired: 3`

```
Start       Old RS = 3   New RS = 0
Step 1      Old RS = 2   New RS = 1   (total: 3)
            ↳ wait until new Pod is Ready
Step 2      Old RS = 1   New RS = 2
            ↳ wait until new Pod is Ready
Final       Old RS = 0   New RS = 3
```

The controller always waits for each new Pod to reach **Ready** before
continuing.

**5. After Successful Rollout**

```
Active RS:  myapp-6f4c9b77d   →   3 × nginx:2.0
Old RS:     myapp-7d6f5d8b7   →   0 pods  (retained for rollback history)
```

### Rollback Flow

**1. User Triggers Rollback**

```bash
kubectl rollout undo deployment myapp
```

**2. Deployment Finds the Previous Revision**

```
Current:   myapp-6f4c9b77d  →  nginx:2.0
Previous:  myapp-7d6f5d8b7  →  nginx:1.0
```

**3. Scaling is Reversed**

```
Start       v2 = 3   v1 = 0
Step 1      v2 = 2   v1 = 1
Step 2      v2 = 1   v1 = 2
Final       v2 = 0   v1 = 3
```

The same rolling mechanism applies — Kubernetes does not modify running
Pods or downgrade containers in-place. Old Pods are recreated from the
previous ReplicaSet; current Pods are terminated.

> **Important:** Rollback is just another rolling update — it reuses the
> existing old ReplicaSet rather than creating a new one.

### Rollback History

By default, Kubernetes retains the last **10** ReplicaSet revisions
(`revisionHistoryLimit`). To inspect:

```bash
kubectl rollout history deployment myapp
kubectl rollout history deployment myapp --revision=2
kubectl rollout undo deployment myapp --to-revision=2   # roll back to a specific revision
```

### Update/Rollback Summary

| Concept | Behaviour |
|---|---|
| `PodTemplateSpec` change | Always creates a new ReplicaSet |
| Old ReplicaSet | Kept at 0 replicas — never deleted (up to `revisionHistoryLimit`) |
| Rollback | Re-scales an existing old RS; does not create a new one |
| Pod mutation | Never — Kubernetes replaces, it does not patch running Pods |
| Rollback mechanism | Identical rolling update logic, just in reverse direction |

### How the ReplicaSet Controller Actually Reconciles

Everything above is the Deployment controller's view: it swaps whole
ReplicaSet objects and shifts replica counts between them. Underneath, each
individual ReplicaSet is kept at its assigned count by the **ReplicaSet
controller**, which is a much dumber, narrower loop than it looks from the
outside.

It watches Pods through a **pod informer**, not just ReplicaSet objects.
Every Pod create/update/delete event is handled by resolving which
ReplicaSet (if any) owns that Pod, and enqueuing **that ReplicaSet's key**
(`namespace/name`) onto a workqueue — never the Pod itself. The workqueue
carries no per-Pod facts; a burst of several Pod-delete events for the same
RS collapses into that one RS key being (re-)enqueued, since the queue
de-dupes.

Ownership is resolved two ways:

- **`ownerReferences`** — a Pod created by this RS already carries a
  controller reference back to it.
- **Label-selector adoption/orphaning** — a Pod with no controller ref but
  matching labels gets adopted (an ownerRef is added). A Pod that stops
  matching the RS's selector (e.g. its labels are edited) gets released —
  ownerRef stripped — but is **not deleted** by the RS. It's simply no
  longer counted. This is also the mechanism behind the classic "isolate
  one Pod for live debugging" trick: `kubectl label pod foo app-` strips it
  from the Deployment/RS without killing it.

When a worker dequeues an RS key, `syncReplicaSet` does a **full recount
from the local informer cache**: list every Pod matching the selector,
filter to non-terminal ones (phase not `Succeeded`/`Failed`, no deletion
timestamp), and compare that count to `.spec.replicas`. There's no
per-event bookkeeping like "this specific Pod died, spawn its replacement"
— every sync recomputes the diff from scratch.

Two consequences worth being explicit about:

- **Readiness is irrelevant to this count.** A `Running` Pod that's
  permanently failing its `readinessProbe` still counts as one of the N
  replicas — the controller has no reason to replace it. Traffic membership
  is a separate concern entirely, owned by the EndpointSlice controller
  (see `pod_lifecycle_and_restarts.md`).
- **The Pod template is consulted only when creating a replacement.** It
  plays no role in evaluating Pods that already exist and still match the
  selector — which is the subject of the next section.

### Why the ReplicaSet Doesn't Enforce Template Conformance on Existing Pods

A natural expectation: if a Pod's spec drifts from `.spec.template`,
shouldn't the RS notice and fix it? It doesn't, and that's deliberate:

1. **Most of the Pod spec is immutable by API-server validation once a Pod
   is admitted** — env vars, volumes, resource requests, scheduling fields,
   and more all reject in-place writes outright. There's very little live
   drift to detect in the first place. `application_configuration.md`'s
   "Pod Spec Immutability" table has the exact mutable-field list:
   `metadata.labels`/`annotations`, `spec.containers[*].image`,
   `spec.tolerations` (additions only), `spec.activeDeadlineSeconds`, and
   CPU/memory via in-place resize.
2. **The fields that are mutable are mutable on purpose**, as operator
   escape hatches — e.g. patching one Pod's image for a live debug session
   without a full rollout. A controller that reverted any deviation from
   the template within milliseconds would make that capability pointless.
3. **Template changes are handled at a different, coarser granularity.** A
   given RS's own template never changes after creation. When the
   Deployment's template changes, the Deployment controller creates an
   entirely new RS object carrying the new template and shifts
   `.spec.replicas` between old and new (the Update Flow above). Conformance
   is enforced by *object substitution*, once, atomically — not by
   continuous per-Pod diffing.
4. **Cost.** Deep-diffing every owned Pod's live spec against the template
   on every reconcile, cluster-wide, is real API-server and CPU load for a
   benefit that's mostly moot given point 1. The ReplicaSet's reconcile
   loop is deliberately the cheapest invariant that does the job:
   `len(list) == int`.
5. **It's relied upon operationally**, not just tolerated — see the
   orphaning trick above. It only works because the RS's ownership/counting
   logic cares exclusively about selector match, nothing else.

The RS's actual guarantee is narrower than it first appears: **N
non-terminal Pods matching the selector**, not **N Pods running this exact
template**. The template only governs Pods the RS itself creates.

### Avoiding Overshoot: the Expectations Mechanism

Every sync recomputes its diff from a **local, potentially stale informer
cache**, not a live read against the API server. Acting on that diff
naively every time a Pod event fires risks overshoot — e.g. issuing a
second batch of creates before the cache has caught up on the first.

The fix is an in-memory per-RS counter called an **expectation**:

1. `syncReplicaSet` recounts and decides it needs, say, 5 more Pods.
2. **Before** issuing any `Create` calls, it records
   `ExpectCreations(rsKey, 5)` — "I'm about to ask for 5 creates; don't
   trust a recount for this RS again until I've observed all 5 land."
3. It issues the 5 `Create` calls, batched via **slow start** (1, then 2,
   then 4... doubling, capped) so a large scale-up doesn't hit the API
   server as one giant burst.
4. At the top of every `syncReplicaSet` call, it checks whether the RS's
   expectations are satisfied. If not, the create/delete decision logic is
   skipped entirely for this sync — only a status update happens.
   Re-enqueues of the same RS key are a no-op on the create/delete side
   until the previous batch is confirmed.
5. The pod informer firing an add event for a new Pod owned by this RS
   decrements the counter by 1. For creates this is a **plain count**, not
   tied to a specific Pod ID — the Pod's name/UID doesn't exist until the
   `Create` call returns.
6. **Deletes are UID-tracked**, unlike creates: `ExpectDeletions(rsKey,
   [uid1..uid5])` records the exact UIDs targeted, since those Pods already
   exist and their UIDs are known before the call goes out. This
   disambiguates "the Pod I asked to be deleted is actually gone" from
   "some unrelated Pod under this RS happened to change right now."
7. If a `Create`/`Delete` call fails synchronously (e.g. a ResourceQuota
   rejection), the expectation is decremented immediately at the failure
   site — the controller doesn't wait for an informer event that will never
   arrive.
8. A several-minute timeout is the fallback safety net for the rare case
   where an expected event genuinely never shows up. It's not the normal
   mechanism — resolution is event-driven and typically near-instant.

### What Triggers Pod Replacement (and What Doesn't)

| Event | RS creates a replacement? | Why |
|---|---|---|
| Pod object deleted (manual, evicted, preempted, node-pressure) | Yes | Active count drops below `.spec.replicas` |
| Pod's labels no longer match the RS selector | Yes (orphaned original keeps running, unmanaged) | No longer counted as active for this RS |
| Container image (or other mutable field) edited directly on a live Pod | No | RS never re-evaluates spec conformance on existing Pods |
| `livenessProbe` failure | No — container restarts in place, same Pod/UID | Kubelet's job, not the RS's; `restartPolicy: Always` is mandatory for controller-owned Pods |
| `readinessProbe` failure | No | Pod is pulled from the Service's EndpointSlice; RS still sees it as `Running`/non-terminal |
| Node-pressure eviction | Yes | Kubelet deletes the whole Pod to protect the Node; same delete-detection path as any other deletion |

## StatefulSet: Stable Identity for Clustered / Data-Bearing Workloads

A StatefulSet gives each replica a **persistent identity keyed off an
ordinal** (0, 1, 2, ...) that survives Pod deletion and recreation. Three
things are pinned to that ordinal:

### Pod identity

The StatefulSet controller creates Pods with deterministic names —
`<sts-name>-<ordinal>` (`postgres-0`, `postgres-1`, `postgres-2`) — not a
random suffix. A replacement Pod for ordinal 0 is always named `postgres-0`.

### Storage identity

Declared via `spec.volumeClaimTemplates`, a StatefulSet-only field (no
Deployment equivalent). For ordinal *N* the controller creates a PVC named
`<template-name>-<sts-name>-<N>`. PVCs are not deleted when their Pod is —
only when the StatefulSet itself is deleted, and even then only if
`persistentVolumeClaimRetentionPolicy` opts into it. So when `postgres-0` is
rescheduled, the controller finds `data-postgres-0` already exists and
reattaches it: same PV, same on-disk data.

This is what makes multi-replica clustered storage possible at all —
Deployments have no per-replica PVC mechanism. A Deployment pointed at one
PVC either fails to schedule replica 2 outright (ReadWriteOnce, different
node → `Multi-Attach error for volume ...: Volume is already exclusively
attached to one node`) or, if the PVC is ReadWriteMany, gets multiple
independent processes racing on the same on-disk files, which most
data-bearing software (Postgres included) actively refuses via its own
locking rather than silently corrupting — Postgres takes an flock-style lock
on `postmaster.pid` at startup and a second instance against the same
directory fails immediately with `FATAL: lock file "postmaster.pid" already
exists ... is another postmaster already running?`. Real Postgres
replication doesn't share a directory at all — each replica is a fully
separate process with its own data directory, populated by streaming WAL
over the network, which is exactly what per-ordinal PVCs + stable DNS names
make possible.

### Network identity

Requires a **headless Service** (`clusterIP: None`) that the StatefulSet
references via `spec.serviceName`. Because there's no cluster IP, CoreDNS
doesn't return one virtual address for the Service — it publishes one DNS A
record per Pod: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`.
The controller sets `pod.spec.hostname`/`pod.spec.subdomain` on each Pod to
make this resolvable.

### Worked example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None        # headless: no VIP, CoreDNS returns per-pod A records instead
  selector:
    app: postgres
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres    # must name the headless Service above
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:     # the field Deployments don't have
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

Deterministic result:

| ordinal | Pod | PVC | DNS name |
|---|---|---|---|
| 0 | `postgres-0` | `data-postgres-0` | `postgres-0.postgres.default.svc.cluster.local` |
| 1 | `postgres-1` | `data-postgres-1` | `postgres-1.postgres.default.svc.cluster.local` |
| 2 | `postgres-2` | `data-postgres-2` | `postgres-2.postgres.default.svc.cluster.local` |

A Patroni-style operator would put `postgres-0.postgres.default.svc.cluster.local`
directly in `postgres-1`'s and `postgres-2`'s `primary_conninfo` — a stable
address that keeps meaning "the pod at ordinal 0" regardless of restarts,
rescheduling, or which node it lands on.

### Startup/shutdown ordering

This is actually two separable guarantees, protecting two different things:

**1. Direction** — Pods are created ascending (0, then 1, then 2 — each
must be Running *and* Ready before the next starts), and deleted descending
(2, then 1, then 0) during scale-down or a rolling update. This protects
**bootstrap dependency**: many clustered systems designate the
lowest-ordinal Pod as a seed/anchor that others contact to join. Ascending
creation guarantees that seed is always up before anything that depends on
it tries to start — and stays true even across a full StatefulSet restart,
since ordinal 0 is always attempted first again.

*Example — Cassandra:* the standard StatefulSet setup points every node's
`seed_provider` at `cassandra-0.cassandra-headless.default.svc.cluster.local`.
A node that starts before the seed is reachable has nothing to gossip with
and can't learn the ring topology. Ordinal-0-first guarantees the seed
exists before any dependent Pod is created.

**2. Cardinality** — the default `podManagementPolicy: OrderedReady` (as
opposed to the alternative, `Parallel`) processes exactly one Pod at a time:
create/delete it, wait for the effect (Ready, or gone), only then touch the
next. This protects **quorum majority** during voluntary operations.

*Example — Zookeeper/etcd, a 3-node ensemble:* Raft-family consensus needs a
majority (2 of 3) reachable to elect a leader or commit a write. If a
rolling update or scale-down took two Pods down simultaneously, majority is
lost mid-operation — no leader, no writes, effectively an outage.
`OrderedReady` guarantees at most one Pod is ever missing at a time, so
majority holds throughout the operation. Setting `podManagementPolicy:
Parallel` on the same ensemble removes this guarantee entirely — two Pods
could be torn down concurrently mid-rollout, taking the ensemble below
quorum.

Neither guarantee is about protecting a specific application role (there is
no general "ordinal 0 is always the primary" rule — Patroni, for instance,
can elect a primary at any ordinal). They're structural: direction protects
against gaps in bootstrap dependency, cardinality protects against losing
majority mid-operation.

## DaemonSet: One Pod Per Node, Not a Replica Count

A DaemonSet has no `spec.replicas` field. "How many Pods" isn't a number
you set — it's implied by cluster topology: exactly one Pod per Node that
matches the DaemonSet's `nodeSelector`/affinity/tolerations, no more, no
less. Used for node-scoped agents — log/metrics collectors, CNI plugins,
`kube-proxy` — where the thing being managed is the node itself, not a
unit of application capacity.

### The reconcile model

The controller runs informers on DaemonSets, Nodes, and Pods — same
List/Watch-against-the-API-server pattern every controller uses; it never
talks to a kubelet directly. For each `(DaemonSet, Node)` pair where the
Node qualifies (passes the DaemonSet's nodeSelector/tolerations), it checks
whether a Pod it owns (via `ownerReferences`) already targets that Node. If
none exists, it creates one; if a qualifying Node stops qualifying (a new
taint added, the Node deleted), it deletes the corresponding Pod. A new
Node joining the cluster needs no manual trigger — the Node informer's add
event drives the reconcile automatically.

### How a Pod actually lands on its node

Since Kubernetes 1.17, the DaemonSet controller does not schedule Pods
itself. It creates a Pod with a hard-coded, single-node node affinity and
hands it to the normal `kube-scheduler`, same path as any other Pod:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchFields:
        - key: metadata.name
          operator: In
          values:
          - node-3
```

`matchFields` against `metadata.name` (a field selector on the Node object)
rather than a label match — deliberately, so it doesn't depend on the
`kubernetes.io/hostname` label being present or correct. There is exactly
one value in that list, always. The scheduler still runs its usual
admission checks against that one Node — resource fit, taints/tolerations —
it just has no other candidate to fall back to.

**This has a real consequence: a DaemonSet Pod that fails to schedule does
not get retried against a different target.** There is no other Node that
satisfies its affinity term — the entire point of the Pod is to be *the*
Pod for node-3. If node-3 lacks capacity or the Pod is missing a required
toleration, the Pod just sits `Pending` until someone fixes the actual
blocker. This is why core DaemonSets like `kube-proxy` ship with
`priorityClassName: system-node-critical` — preemption, not retry-elsewhere,
is the mechanism that reclaims room for them.

### Taints don't get bypassed automatically

A control-plane Node typically carries `node-role.kubernetes.io/control-plane:NoSchedule`
specifically to keep ordinary workload Pods off it. A log collector that also
needs to run there needs an explicit toleration in its own Pod spec — nothing
about being a DaemonSet exempts it:

```yaml
tolerations:
  - key: node-role.kubernetes.io/control-plane
    effect: NoSchedule
    operator: Exists
```

### Cordon and drain

`kubectl cordon` sets `Node.spec.unschedulable`, which blocks *new* ordinary
Pods from landing — but the DaemonSet controller's own eligibility check
ignores that field, by design, since agents like `kube-proxy` need to keep
running on a Node you're about to drain, not disappear from it the moment
it's cordoned. That's the reason `kubectl drain --ignore-daemonsets` exists
(already used in `kubernetes_scenarios.md`'s node-replacement scenario, without
explanation there): evicting a DaemonSet Pod would be pointless churn — the
controller would immediately recreate an identical Pod pinned to that same
still-existing Node. `drain` just leaves them running until the Node object
itself is deleted.

### Node deletion

Covered in full in `scheduling.md`'s Pod Garbage Collection section — the
short version: once a Node object is deleted, the PodGC controller (not the
DaemonSet controller) removes the orphaned Pod that was bound to it. The
DaemonSet controller doesn't create a replacement anywhere, because a
deleted Node was the only valid target that Pod ever had.

## The Decision Rule

For Deployment vs StatefulSet: not "stateless vs stateful" — plenty of
stateful software runs fine on a Deployment with one PVC and one replica; it
just isn't *clustered*. The actual condition: **use a StatefulSet when
replicas need to address or distinguish each other by a stable, persistent
identity that survives rescheduling** — a leader that replicas must
reconnect to by name, a consensus ensemble whose members are statically
configured by address, a shard whose data is pinned to a specific instance.
Use a Deployment when every replica is interchangeable and nothing — not a
peer, not a client, not the app itself — needs to know or care which
specific instance handled the last request.

DaemonSet is orthogonal to that whole axis. It isn't "in between" Deployment
and StatefulSet on an identity spectrum — it answers *where*, not *how many*
or *whether interchangeable*: run exactly one instance per matching Node,
forever, regardless of any desired count. Reach for it when the workload's
job is intrinsically about the Node it runs on, not about serving
application traffic.

## What This Covers So Far

This covers Deployment's Pod-selection ranking, update/rollback flow across
ReplicaSets, and the ReplicaSet controller's own reconcile loop (pod
informer, ownerRef/selector adoption and orphaning, the expectations
mechanism that prevents overshoot, and why it doesn't enforce template
conformance on Pods that already exist); StatefulSet's three identity
mechanisms (Pod naming, per-ordinal PVCs via `volumeClaimTemplates`, per-Pod
DNS via a headless Service) and why startup/shutdown ordering matters, split
into the direction guarantee (bootstrap dependency) and the cardinality
guarantee (quorum preservation); and DaemonSet's reconcile model (no replica
count, per-node fixed node-affinity pinning via the normal scheduler, why a
failed schedule never retries elsewhere, the cordon/drain exemption, and
Node-deletion cleanup).

Not yet covered: the exact slow-start batch sizes/cap for ReplicaSet Pod
creation, and whether they differ for creates vs. deletes; how the
Deployment controller's own reconcile loop decides step-by-step
surge/unavailable counts across old and new RS during a rolling update —
this doc shows the end states, not the per-step decision logic; how
failover actually works once a StatefulSet primary dies (how a
Patroni-style operator detects it and repoints the other replicas — the
`primary_conninfo` mechanism above is only half the story);
`persistentVolumeClaimRetentionPolicy` and what happens to PVCs when a
StatefulSet itself (not just a Pod) is deleted or scaled down; partitioned
rolling updates via `spec.updateStrategy.rollingUpdate.partition`
(StatefulSet's canary-style mechanism, no Deployment equivalent); how a
PodDisruptionBudget interacts with `OrderedReady` during *involuntary*
disruption (a node drain), and how it gates the Eviction API path generally
across all three controllers; DaemonSet's own update strategy
(`RollingUpdate` vs `OnDelete`, `maxUnavailable`) — no ordinal concept at
all, worth contrasting directly against StatefulSet's ordering; static Pods
(kubelet-managed manifests), a mechanism people sometimes confuse with
DaemonSets but which doesn't go through the API server's scheduling path at
all; and in-place Pod vertical scaling's interaction with the ReplicaSet
expectations mechanism — does a resize count as a "create" or "delete"
expectation, or neither?
