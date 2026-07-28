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

Full create/update/rollback mechanics (surge, `maxUnavailable`, revision
history) are covered in `deployment_replicasets.md` — this section only
covers what's relevant to the identity comparison.

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

This covers Deployment's Pod-selection ranking; StatefulSet's three identity
mechanisms (Pod naming, per-ordinal PVCs via `volumeClaimTemplates`, per-Pod
DNS via a headless Service) and why startup/shutdown ordering matters, split
into the direction guarantee (bootstrap dependency) and the cardinality
guarantee (quorum preservation); and DaemonSet's reconcile model (no replica
count, per-node fixed node-affinity pinning via the normal scheduler, why a
failed schedule never retries elsewhere, the cordon/drain exemption, and
Node-deletion cleanup).

Not yet covered: how failover actually works once a StatefulSet primary
dies (how a Patroni-style operator detects it and repoints the other
replicas — the `primary_conninfo` mechanism above is only half the story);
`persistentVolumeClaimRetentionPolicy` and what happens to PVCs when a
StatefulSet itself (not just a Pod) is deleted or scaled down; partitioned
rolling updates via `spec.updateStrategy.rollingUpdate.partition`
(StatefulSet's canary-style mechanism, no Deployment equivalent); how a
PodDisruptionBudget interacts with `OrderedReady` during *involuntary*
disruption (a node drain) rather than the voluntary rollouts/scale-downs
discussed here; DaemonSet's own update strategy (`RollingUpdate` vs
`OnDelete`, `maxUnavailable`) — no ordinal concept at all, worth contrasting
directly against StatefulSet's ordering; and static Pods (kubelet-managed
manifests), a mechanism people sometimes confuse with DaemonSets but which
doesn't go through the API server's scheduling path at all.
