# Kubernetes Scheduling, Preemption, and Eviction

## Scheduling

The Kubernetes scheduler (`kube-scheduler`) is responsible for assigning Pods to Nodes. It runs as part of the control plane and watches for newly created Pods that have no Node assigned, then selects the best Node for each Pod.

### Scheduling Cycle

Scheduling happens in two phases:

1. **Filtering** — eliminates Nodes that do not satisfy the Pod's requirements (resource requests, node selectors, taints/tolerations, affinity rules, etc.)
2. **Scoring** — ranks the remaining Nodes using a set of priority functions; the Node with the highest score wins

If no Node passes the filtering phase, the Pod remains `Pending`.

### Binding: Making the Decision Permanent

Scoring picks a winning Node, but the scheduler doesn't just note that
internally — it makes one explicit API call: a `POST` to the Pod's
`binding` subresource, naming the target Node. The API server turns that
into setting `pod.spec.nodeName`. This happens exactly once per Pod object,
and the field is immutable afterward — nothing in Kubernetes ever moves a
running Pod from one Node to another.

Any apparent "rescheduling" you've seen elsewhere isn't the same Pod
relocating. When a ReplicaSet-owned Pod ends up on a different Node after
its original one failed, that's the old Pod object being deleted and a
brand-new one — new name, new UID, no `nodeName` yet — created by the
owning controller, which independently goes through this exact same
filter → score → bind cycle and can land anywhere that fits. The Pod
identity changed; only the *replica* persisted.

### Influencing Scheduling

| Mechanism | Description | Example |
|---|---|---|
| `nodeSelector` | Simple key/value label match on the Node | `nodeSelector: {disk: ssd}` to keep a database Pod off spinning-disk Nodes |
| `nodeName` | Direct assignment to a specific Node, bypasses the scheduler | Pinning a one-off debug Pod to a specific Node to reproduce a Node-local issue (a stuck kubelet, a corrupt image cache) |
| Node Affinity | Expressive label-based rules (`requiredDuringScheduling`, `preferredDuringScheduling`) | `preferredDuringSchedulingIgnoredDuringExecution` weighting Nodes in `zone=us-east-1a`, without hard-failing if none are free |
| Pod Affinity / Anti-Affinity | Co-locate or spread Pods relative to other Pods | Anti-affinity on `topologyKey: kubernetes.io/hostname` so no two replicas of the same Deployment land on the same Node |
| Taints & Tolerations | Nodes repel Pods unless the Pod explicitly tolerates the taint | Tainting GPU Nodes `nvidia.com/gpu=true:NoSchedule` so only ML workloads with the matching toleration land there |
| Topology Spread Constraints | Distribute Pods evenly across zones, nodes, or custom topologies | `maxSkew: 1` over `topology.kubernetes.io/zone` to keep replica count within 1 of each other across zones |
| Resource Requests & Limits | `requests` drive scheduling decisions; `limits` cap runtime usage | See Scoring Strategies below for how `requests` alone (not `limits`) feed the bin-packing/spreading math |

**Two gotchas worth flagging, since both mechanisms above look similar to a Node's other placement rules but behave differently once a Pod is already running:**

- **`nodeName` skips filtering entirely, not just scoring.** Every other row in the table above is enforced by the scheduler's filtering phase — including the resource-request check against a Node's free capacity. `nodeName` hands the Pod straight to a specific Node's kubelet, so that capacity check never happens. An over-scheduled Node here doesn't leave the Pod `Pending` the way a failed filter would; the kubelet just tries to run it, which is one of the more common (and self-inflicted) ways a Node ends up crossing the eviction thresholds described in Node-Pressure Eviction below.
- **Affinity is `...IgnoredDuringExecution`; a `NoExecute` taint is not.** Every affinity/anti-affinity rule Kubernetes ships today only applies at scheduling time — if a Node's labels change after the Pod is already running (say, a `zone` label gets corrected), the scheduler does not go back and re-evaluate; the Pod stays put. Taints are the exception: a `NoExecute` taint added to a Node *after* the fact will actively evict Pods that lack a matching toleration, immediately. It's a third, distinct removal path alongside the two covered under Eviction below — neither API-initiated (no `kubectl drain` involved) nor Node-pressure driven (the Node can be perfectly healthy), just a policy decision propagating live.

### Scoring Strategies — Bin-Packing vs. Spreading

Filtering narrows the field to feasible Nodes; scoring then decides which one wins. When every Node passes filtering with equal resource availability, the deciding factor is the **`scoringStrategy`** of the `NodeResourcesFit` plugin (part of the kube-scheduler's default profile). There are three options.

**Setup for the examples below:** 2 Nodes, each with 8 CPU allocatable. Node A is empty. Node B already runs Pods requesting 4 CPU total. A new Pod requesting 2 CPU needs to be placed.

#### `LeastAllocated` (default)

Favors the Node with the **most free capacity** — this spreads Pods evenly across the cluster.

```
score = ((capacity - requested) / capacity) * 100
```

```
Node A: (8 - 2) / 8 * 100 = 75   ← includes the new Pod's own request
Node B: (8 - 6) / 8 * 100 = 25
```

Node A wins → **the new Pod lands on the empty Node.** This is why, absent any other config, freshly created Pods tend to fan out across Nodes rather than pile onto ones that already have workload.

Use this for general-purpose workloads: it keeps headroom free on every Node for traffic bursts and reduces noisy-neighbor risk.

#### `MostAllocated`

Favors the Node with the **highest existing utilization** — this bin-packs Pods onto fewer Nodes.

```
score = (requested / capacity) * 100
```

```
Node A: 2 / 8 * 100 = 25
Node B: 6 / 8 * 100 = 75
```

Node B wins → **the new Pod lands on the Node that already has Pods.**

Use this when the goal is consolidation rather than spread — e.g. packing Pods tightly so the Cluster Autoscaler can identify and scale down empty Nodes, which matters for cost efficiency on cloud clusters.

#### `RequestedToCapacityRatio`

A generalized version of the above two — you supply a `shape` (a utilization % → score curve) and the plugin scores Nodes against it. It has no fixed default behavior; it reproduces whichever strategy the shape encodes:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: RequestedToCapacityRatio
        resources:
        - name: cpu
          weight: 1
        requestedToCapacityRatio:
          shape:
          - utilization: 0
            score: 0
          - utilization: 100
            score: 10
```

- Shape `(0 → 0, 100 → 10)` — higher utilization scores higher → **behaves like `MostAllocated`** (bin-packing).
- Shape `(0 → 10, 100 → 0)` — inverted → **behaves like `LeastAllocated`** (spreading).
- A shape that peaks in the middle (e.g. `0 → 0, 50 → 10, 100 → 0`) avoids both near-empty and near-full Nodes — useful for keeping headroom for larger Pods while still discouraging fragmentation across too many Nodes.

This is a **cluster-wide scheduler config**, set in the `KubeSchedulerConfiguration`, not something an individual Pod spec can request.

> **Note:** `NodeResourcesFit` is rarely the only scoring plugin in play. `PodTopologySpread` (a soft, default constraint for Pods owned by a Deployment/ReplicaSet/StatefulSet/Service) and `ImageLocality` (favors Nodes that already have the image cached) also contribute to the final score at weight 1 each. In the example above, both strategies still land the Pod on Node A/B respectively, but real clusters will see all default-weighted plugins summed together.

---

## Preemption

When a high-priority Pod cannot be scheduled because all Nodes are full, the scheduler may **preempt** (evict) lower-priority Pods to free up capacity.

### Priority Classes

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
```

Pods reference a PriorityClass via `spec.priorityClassName`. Built-in system classes (`system-cluster-critical`, `system-node-critical`) have the highest values and protect core components.

### Preemption Flow

1. The scheduler identifies the pending high-priority Pod.
2. It finds Nodes where evicting lower-priority Pods would make room.
3. It selects the Node that minimizes disruption (fewest victims, lowest priority sum).
4. Victim Pods receive a graceful termination signal and are deleted.
5. The high-priority Pod is then scheduled onto the freed Node.

### Disruption Budget Interaction

A `PodDisruptionBudget` (PDB) limits voluntary disruptions. The scheduler respects PDBs during preemption — if evicting a Pod would violate its PDB, the scheduler looks for other candidates first. However, preemption can still violate a PDB as a last resort when no other option exists.

---

## Eviction

Eviction removes running Pods from a Node. There are two distinct types.

### API-Initiated Eviction

Triggered by a client (e.g., `kubectl drain`) via the Eviction API. It is a voluntary, graceful removal that respects PodDisruptionBudgets. Used for node maintenance and rolling upgrades.

```bash
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
```

### PodDisruptionBudget: How "Voluntary" Is Actually Checked

The reason PDBs exist at all: a node drain, a cluster-autoscaler scale-down,
or a manual eviction are all **proactive, plannable** actions — unlike a
node crashing or an OOM kill, something initiates them deliberately, which
means something can also be made to check first whether doing so is safe.
That check is what a PDB is.

The mechanism is narrower than "any voluntary pod removal," though: a PDB is
enforced **only** at the Eviction API — the `pods/eviction` subresource.
`kubectl drain`, cluster-autoscaler, and any other caller that goes through
that subresource get checked against the PDB's `status.disruptionsAllowed`;
a request that would push a matching group of Pods below the budget is
rejected outright with HTTP `429`.

A Deployment's own ReplicaSet controller scaling down old Pods during a
rolling update does **not** go through this path — it's a direct Pod
`delete`, the same call used in the `What Triggers Pod Replacement` table in
`workloads.md`. **PDBs are never consulted on that path.** A rollout's pace
is governed solely by its own `maxSurge`/`maxUnavailable`; no PDB throttles
it, however tight.

`status.disruptionsAllowed` (and `currentHealthy`/`desiredHealthy` alongside
it) isn't a ledger of who's allowed to disrupt what — it's continuously
recomputed by the disruption controller from **live Pod health**: how many
Pods matching the PDB's selector are currently `Ready`, compared against the
budget. It has no memory of *why* a Pod became unavailable.

That's what makes this scenario possible: a Deployment rollout and a node
drain targeting the same PDB-selected Pods, running at the same time, on
different paths.

```
Rollout in progress: RS deletes old Pods directly (not through eviction)
  → currentHealthy drops → disruptionsAllowed hits 0
      (the PDB has no idea this drop came from a rollout, not a disruption)

Concurrent node drain: evicts a *different* Pod under the same PDB
  → Eviction API checks disruptionsAllowed → sees 0 → rejects with 429
  → kubectl drain / cluster-autoscaler retries with backoff

Rollout's new Pods reach Ready → currentHealthy recovers →
disruptionsAllowed climbs back above 0 → the next eviction retry succeeds
```

The drain isn't stuck forever — it resolves once the rollout converges and
Pod health recovers — but it stalls for the duration of that overlap,
entirely because the PDB is reading live health, not distinguishing which
controller caused the dip.

### Node-Pressure Eviction (Kubelet-Initiated)

The `kubelet` monitors Node resource consumption and evicts Pods when thresholds are crossed to protect Node stability.

**Eviction signals:**

| Signal | Description |
|---|---|
| `memory.available` | Available memory on the Node |
| `nodefs.available` | Available disk on the root filesystem |
| `nodefs.inodesFree` | Free inodes on the root filesystem |
| `imagefs.available` | Available disk on the image filesystem |

**Eviction thresholds** are configured in the kubelet (e.g., `--eviction-hard=memory.available<100Mi`).

**Eviction order** — the kubelet ranks Pods for eviction as follows:

1. Pods exceeding their resource requests (`BestEffort` QoS first)
2. `Burstable` Pods that exceed their requests
3. `Guaranteed` Pods (evicted last, only under extreme pressure)

### QoS Classes and Eviction

| QoS Class | Condition | Eviction Priority |
|---|---|---|
| `BestEffort` | No requests or limits set | Evicted first |
| `Burstable` | Requests set but < limits, or only some containers have them | Evicted second |
| `Guaranteed` | Requests == limits for all containers | Evicted last |

---

## Pod Garbage Collection

A fourth removal path, distinct from the three covered above (API-initiated
eviction, node-pressure eviction, and the `NoExecute`-taint path flagged in
the gotchas under Influencing Scheduling): the **PodGC controller**, part of
`kube-controller-manager`, cleans up Pods whose bound Node no longer exists
as an API object.

Because `pod.spec.nodeName` is immutable (see Binding, above), deleting a
Node doesn't touch any Pod that was bound to it — the Pod object just sits
there, still claiming `nodeName: worker-3`, pointing at a Node that's gone.
Nothing will ever update its status again: the kubelet that owned it no
longer exists to report anything or run a graceful termination. PodGC's
reconcile loop lists all Pods, checks each one's `nodeName` against the live
Node set from its own Node lister, and force-deletes — skipping graceful
termination entirely, since there's no kubelet left to ask — any Pod whose
Node is missing.

This is a general-purpose mechanism, not aware of what (if anything) owns
the Pod. For a ReplicaSet-owned Pod, the ReplicaSet notices the drop in
Ready count once PodGC removes the orphan, and creates a fresh replacement
that the scheduler binds to a live Node via the normal cycle above. For a
DaemonSet-owned Pod there's no equivalent replacement: the DaemonSet
controller treats a deleted Node as one fewer place a Pod should exist at
all, so once PodGC clears the orphan, nothing recreates it anywhere — see
`workloads.md`'s DaemonSet section for the full one-pod-per-node reconcile
model this fits into.

PodGC has two other, unrelated responsibilities worth knowing exist even
though they're out of scope here: capping the number of terminated
(`Succeeded`/`Failed`) Pods kept around per Node
(`--terminated-pod-gc-threshold`), and cleaning up Pods stuck `Terminating`
on a Node marked with the `out-of-service` taint (non-graceful Node
shutdown handling).

---

## Key Interactions

- **Preemption vs. Eviction** — preemption is scheduler-driven to place a pending Pod; eviction is kubelet-driven to protect a running Node.
- **PriorityClass affects both** — higher-priority Pods are harder to preempt and harder to evict under node pressure.
- **PodDisruptionBudgets** gate the Eviction API path specifically (drains, autoscaler scale-down) — not a controller's own direct Pod deletes (e.g. a Deployment rollout scaling down its old ReplicaSet) — and provide only a soft guarantee during node-pressure eviction.
- **Resource requests matter** — they are the primary input to both the scheduling decision and the kubelet's eviction ordering.
- **Pod GC vs. the other removal paths** — eviction and preemption remove a Pod from a Node that still exists, expecting a replacement (if any) to land elsewhere; Pod GC cleans up after the Node itself is gone. `nodeName`'s immutability is exactly why that cleanup has to be a deletion, not a reassignment.

## What This Covers So Far

This covers the scheduling cycle (filtering, scoring, binding) and the
mechanisms that influence it, the three `NodeResourcesFit` scoring
strategies, preemption and its PDB interaction, the two standard eviction
paths plus the `NoExecute`-taint path, PodDisruptionBudget's actual
enforcement point (the Eviction API subresource, not a controller's direct
Pod deletes) and its live-health-based `disruptionsAllowed` recompute — including
the concrete case of a Deployment rollout and a node drain contending over
the same PDB — and Pod GC's role in cleaning up Pods orphaned by Node
deletion.

Not yet covered: scheduler extenders and running multiple schedulers in one
cluster; the full default scoring plugin set beyond `NodeResourcesFit`,
`PodTopologySpread`, and `ImageLocality`; the descheduler (a separate,
non-core project that actively rebalances already-*running* Pods — distinct
from everything here, which only acts at specific trigger points: creation,
pressure, or an explicit request); PodGC's other two responsibilities
(terminated-Pod capping, `out-of-service` taint handling) in any real
detail; and how a PDB's `OrderedReady` interaction plays out for a
StatefulSet under *involuntary* disruption specifically (a node drain hitting
a quorum-sensitive ensemble), as opposed to the voluntary-vs-voluntary
rollout/drain race covered above.
