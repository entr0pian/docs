# Kubernetes Scenarios

| # | Scenario | The question |
|---|---|---|
| 1 | [Replace an Unresponsive Worker Node](#scenario-replace-an-unresponsive-worker-node) | Replace a dead node without affecting application availability |
| 2 | [Ensure Data Persistence for a Stateful Application](#scenario-ensure-data-persistence-for-a-stateful-application) | Keep a stateful app's data when its Pods are rescheduled or updated |
| 3 | [Pod Stuck in Pending State](#scenario-pod-stuck-in-pending-state) | Possible reasons a Pod stays `Pending`, and how to troubleshoot it |
| 4 | [Requests, Limits, QoS and the HPA](#scenario-requests-limits-qos-and-the-hpa) | QoS class, HPA replica math, and the cost of setting requests = limits |
| 5 | [`kubectl drain` Blocked by a PodDisruptionBudget](#scenario-kubectl-drain-blocked-by-a-poddisruptionbudget) | Why the drain is stuck on 429s, the root causes, and how to unblock it safely |
| 6 | [QoS and Pod Priority Under Memory Pressure](#scenario-qos-and-pod-priority-under-memory-pressure) | Who kills which pod (scheduler, kubelet, kernel), and what each one reads |
| 7 | [Upgrading a Production EKS Cluster](#scenario-upgrading-a-production-eks-cluster) | Pre-checks, order and skew, removed APIs, and a webhook outage during an irreversible upgrade |
| 8 | [Healthy Pods, Intermittent 504s](#scenario-healthy-pods-intermittent-504s) | Pods are Running and Ready but users get intermittent 504s: who generated it, and why |

---

## Scenario: Replace an Unresponsive Worker Node

> **Question:** You have a Kubernetes cluster with multiple worker nodes. One of the nodes becomes unresponsive and needs to be replaced. Explain the steps you would take to replace the node without affecting the availability of applications running on the cluster.

> **Goal:** Maintain application availability while safely removing and replacing an unhealthy node.

### Key Mechanisms Involved

- `cordon` / `drain`
- ReplicaSets / Deployments
- Readiness probes + `preStop` hooks
- PodDisruptionBudgets (PDBs)
- Cluster Autoscaler / node provisioning

---

### Step 1 — Assess Node State

Determine whether the node is partially unhealthy or fully unreachable, and which workloads are affected.

```bash
kubectl get nodes
kubectl describe node worker-3
kubectl get pods -A -o wide | grep worker-3
```

Common node statuses: `Ready`, `NotReady`, `Unknown`.

Diagnose the root cause — infra, networking, disk, or kubelet — before taking action.

---

### Step 2 — Cordon (Prevent New Scheduling)

```bash
kubectl cordon worker-3
```

Marks the node `unschedulable`. Existing Pods continue running — this stops the situation getting worse while you work.

---

### Step 3 — Verify Cluster Capacity

**This is the step most teams skip — and where outages happen.**

Before draining, confirm the remaining nodes can absorb the evicted workloads. Check:

- Remaining CPU/memory across nodes (`kubectl top nodes`)
- Anti-affinity and topology spread constraints
- PDBs that may block eviction
- Stateful workloads with local storage

If headroom is insufficient: scale the node group first, let the Cluster Autoscaler add capacity, or manually provision a replacement node before draining.

---

### Step 4 — Drain the Node

```bash
kubectl drain worker-3 \
  --ignore-daemonsets \
  --delete-emptydir-data
```

Drain evicts Pods, respects PDBs, and waits for graceful termination. ReplicaSets/Deployments recreate evicted Pods on healthy nodes automatically.

**What happens internally (example: Deployment with 3 replicas, one Pod on the draining node):**

```
Drain triggers eviction
  → Pod receives SIGTERM
  → Readiness probe fails → endpoint removed from Service
  → Traffic stops reaching the Pod
  → Deployment/ReplicaSet creates replacement Pod on another node
  → New Pod becomes Ready
  → Old Pod terminates
```

---

### Production Reliability Considerations

#### Readiness Probes

Without a readiness probe, traffic keeps reaching a terminating Pod. A proper probe ensures the Pod is removed from Service endpoints before shutdown begins.

#### `preStop` Hook + Grace Period

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "20"]
terminationGracePeriodSeconds: 40
```

This gives kube-proxy, ingress controllers, and load balancers time to drain connections before the process exits. The `preStop` sleep must fit within `terminationGracePeriodSeconds`.

#### PodDisruptionBudgets

```yaml
minAvailable: 2
```

`drain` respects PDBs — it will block eviction if it would violate the budget. Without PDBs, a drain can take down multiple replicas simultaneously, causing downtime.

---

### Step 5 — Handle Special Workloads

Not all workloads are stateless Deployments. Pay extra attention to:

| Workload | Concern |
|---|---|
| **StatefulSets** | Storage reattachment, quorum, replication health |
| **DaemonSets** | Skipped by drain (`--ignore-daemonsets`) — verify agents restart on new node |
| **Local PVs** | Data is lost on eviction — must be handled explicitly |
| **Singleton / leader-elected workloads** | Ensure leader re-election completes cleanly |
| **Kafka / database nodes** | Validate partition rebalancing and replica health before eviction |

---

### Step 6 — Remove and Replace the Node

```bash
kubectl delete node worker-3
```

Then reprovision depending on infrastructure:

- **EKS managed node group** — terminate the EC2 instance; ASG replaces it automatically
- **Cluster Autoscaler** — deletes the node object; CA handles reprovisioning
- **Cluster API** — delete the `Machine` resource
- **Bare metal** — reprovision via your provisioning tooling

---

### Step 7 — Validate Cluster Recovery

```bash
kubectl get nodes
kubectl get pods -A
kubectl get events
```

Verify:
- All Pods rescheduled and `Ready`
- No `Pending` or `CrashLoopBackOff` Pods
- No PDB violations
- New node joined and is `Ready`
- DaemonSet agents (logging, monitoring) running on new node

Also validate through monitoring: ingress traffic, latency, error rates, and Cluster Autoscaler stabilization.

---

### Edge Case: Node Fully Unreachable

If kubelet is dead, `kubectl drain` may hang — the API server cannot evict Pods on an unreachable node. Pods may stay stuck in `Terminating`.

Force deletion as a last resort:

```bash
kubectl delete pod <pod-name> --force --grace-period=0
```

Use with caution — especially for StatefulSets — as it risks:
- Split-brain (two instances of the same Pod running simultaneously)
- Duplicate writes
- Storage corruption if a PV is reattached before the old Pod is truly gone

---

### Senior-Level Insight

The real goal is not "replace the node" — it is **maintaining service availability during infrastructure failure**.

Kubernetes provides the orchestration primitives, but availability depends on how well the applications are designed for it:

| Concern | Mechanism |
|---|---|
| Traffic continuity | Readiness probes + endpoint removal |
| Graceful shutdown | `preStop` + `terminationGracePeriodSeconds` |
| Replica availability | PDBs + sufficient replica count |
| Topology resilience | Anti-affinity + topology spread constraints |
| Capacity headroom | Cluster Autoscaler + node group sizing |

A well-designed platform should tolerate node loss as a **routine operation**, not an emergency.

### Summary

When a node becomes unresponsive, my first instinct is not to drain it immediately — it is to assess what is actually wrong and whether the cluster has enough capacity to absorb the workloads. Draining without capacity headroom is how teams cause the outage they were trying to avoid.

The sequence I follow is: cordon the node first to stop new scheduling, verify the remaining nodes can take the load, then drain — which respects PodDisruptionBudgets and waits for graceful termination. Once drained, I delete the node object and let the infrastructure layer replace it — whether that is an EKS managed node group, a Cluster Autoscaler-triggered scale-up, or a manual reprovisioning step.

The part most answers miss is that availability during this operation depends almost entirely on how well the applications are built for it: readiness probes to drain endpoints before shutdown, `preStop` hooks to give load balancers time to catch up, and PDBs to prevent multiple replicas going down simultaneously. Without those, Kubernetes will technically do the right thing at the orchestration level, but requests will still fail.

If the node is fully unreachable, drain may hang and Pods stay stuck in `Terminating`. Force deletion is the escape hatch, but I use it carefully — for StatefulSets especially, it risks split-brain and storage corruption if the volume gets reattached before the old Pod is truly gone.

---

## Scenario: Ensure Data Persistence for a Stateful Application

> **Question:** You have a stateful application running on Kubernetes that requires persistent storage. How would you ensure that the data is retained when the pods are rescheduled or updated?

> **Goal:** Decouple Pod lifecycle from data lifecycle — Pods are ephemeral, but data must survive rescheduling, updates, and node failures.

### Key Principle

```
Pod dies       →  data survives
Pod reschedules →  data reattaches
Deployment rolls →  data persists
```

The solution is built on: **PersistentVolumes**, **PersistentVolumeClaims**, **StatefulSets**, and correct reclaim policies.

---

### Storage Model

```
Pod
 └─ mounts PVC
      └─ bound to PV
           └─ backed by actual storage (EBS, EFS, Ceph, local SSD, ...)
```

The critical property: **PVC and PV outlive the Pod**. When a Pod is rescheduled, Kubernetes detaches the volume from the old node and reattaches it to the new one. The application resumes with existing data intact.

---

### StatefulSet vs Deployment for Stateful Workloads

For databases and clustered systems, use a **StatefulSet** rather than a Deployment.

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod identity | Random hash suffix | Stable ordinal (`mydb-0`, `mydb-1`) |
| Storage | Shared or none | Each Pod gets its own PVC |
| Rollout order | Parallel | Sequential (ordered) |
| Use case | Stateless services | Databases, Kafka, Elasticsearch, ZooKeeper |

Stable identity matters for systems that use Pod name for cluster membership or replication config.

---

### StatefulSet + `volumeClaimTemplates`

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: ["ReadWriteOnce"]
    storageClassName: gp3
    resources:
      requests:
        storage: 100Gi
```

When the StatefulSet creates `mydb-0`, Kubernetes automatically creates PVC `data-mydb-0` and binds it to a PV. Each Pod gets its own dedicated volume:

```
mydb-0  →  data-mydb-0
mydb-1  →  data-mydb-1
mydb-2  →  data-mydb-2
```

This association is preserved across restarts and rescheduling.

---

### What Happens When a Pod is Rescheduled

```
mydb-0 terminated (node failure / eviction)
  → PVC data-mydb-0 remains untouched
  → StatefulSet recreates mydb-0 on another node
  → Volume detached from old node, reattached to new node
  → Pod mounts same storage
  → Application starts with existing data
```

---

### StorageClass Considerations

Storage backend behavior affects scheduling strategy:

| Backend | Behaviour |
|---|---|
| **AWS EBS** | AZ-bound — Pod must reschedule within the same AZ |
| **AWS EFS** | Multi-AZ, multi-node shared mounts (`ReadWriteMany`) |
| **Local volumes** | Node-bound — Pod pinned to that node via `nodeAffinity` |
| **Ceph / NetApp** | Network-attached — flexible scheduling |

For EBS, use topology-aware provisioning (`WaitForFirstConsumer` binding mode) so the volume is created in the same AZ as the scheduled Pod.

---

### Reclaim Policy

```yaml
persistentVolumeReclaimPolicy: Retain
```

For production databases, always use **Retain**. Deleting a PVC should never silently destroy production data. With `Delete` (the default for dynamic provisioning), the underlying disk is deleted when the PVC is removed.

---

### Update / Rollout Behavior

StatefulSet updates are sequential and ordered — highest ordinal first:

```
mydb-2 updated → healthy
mydb-1 updated → healthy
mydb-0 updated → healthy
```

This prevents quorum loss and cluster instability during rolling updates. The `maxUnavailable` field controls how many Pods can update in parallel (default: 1).

---

### Production Reliability Considerations

#### Backups

Persistence protects from Pod loss. It does **not** protect from:
- Data corruption
- Accidental deletion
- Ransomware / operator mistakes

Implement in addition to PVs:
- Scheduled volume snapshots
- Logical backups (pg_dump, mysqldump)
- PITR (Point-In-Time Recovery) where supported
- Cross-region backup replication

#### PodDisruptionBudgets

```yaml
minAvailable: 2
```

Prevents multiple database replicas going down simultaneously during node drains, upgrades, or autoscaling events.

#### Anti-Affinity / Topology Spread

Distribute replicas across nodes and availability zones to avoid a single failure domain taking down the whole cluster:

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
```

#### Graceful Shutdown

Databases need time to flush WAL, close transactions, and leave cluster membership cleanly:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 15"]
terminationGracePeriodSeconds: 60
```

Configure both readiness and liveness probes so Kubernetes correctly tracks application health independently of Pod health.

---

### Senior-Level Insight

Persistent storage in Kubernetes is not just "attach a disk." It requires coordinating orchestration, scheduling, storage lifecycle, and failure handling.

| Concern | Mechanism |
|---|---|
| Data survives Pod loss | PVC/PV decoupled from Pod lifecycle |
| Stable identity for clustering | StatefulSet ordinal names |
| No accidental data deletion | `Retain` reclaim policy |
| AZ-aware scheduling | `WaitForFirstConsumer` + topology constraints |
| Rolling update safety | StatefulSet sequential rollout + PDBs |
| Disaster recovery | Snapshots + logical backups + PITR |

The platform ensures storage survives Pod lifecycle. The application must still be designed for replication, consistency, recovery, and failover — Kubernetes provides the infrastructure, not the application-level guarantees.

### Summary

The core principle is simple: decouple the Pod lifecycle from the data lifecycle. Pods are ephemeral — data must not be.

The mechanism is a PVC bound to a PV that outlives the Pod. When a Pod is rescheduled, Kubernetes detaches the volume from the old node and reattaches it to the new one. The application comes back up with the same data.

For anything beyond a single stateless instance, I use a StatefulSet rather than a Deployment. The reason is stable identity: `mydb-0`, `mydb-1`, `mydb-2` persist across rescheduling, and each Pod keeps its own dedicated PVC. That stable name-to-storage mapping is what systems like Kafka, PostgreSQL, and Elasticsearch depend on for cluster membership and replication.

A few things I always make explicit: storage backend constraints matter — EBS is AZ-bound, so topology-aware provisioning is required to avoid Pods being scheduled into the wrong zone. The reclaim policy should be `Retain` in production — `Delete` is the dynamic provisioning default and will silently destroy data when a PVC is removed.

Finally, persistence alone is not enough. PVs protect against Pod loss. They do not protect against corruption, accidental deletion, or operator mistakes. Backups, snapshots, and PITR are a separate layer that has to exist alongside the storage configuration.

---

## Scenario: Pod Stuck in Pending State

> **Question:** A Kubernetes pod is stuck in a "Pending" state. What could be the possible reasons, and how would you troubleshoot it?

> **Goal:** Identify whether the block is at the scheduler level or the kubelet/runtime level, and resolve the underlying infrastructure or configuration issue.

---

### What Pending Actually Means

A Pod in `Pending` means the API server accepted the Pod object, but Kubernetes has not been able to fully place or start it yet.

```
Important distinction:
  Pending ≠ container crash
  Pending ≠ application issue
```

It is almost always a **scheduling or infrastructure dependency problem**.

**Simplified Pod lifecycle — Pending happens before Running:**

```
Pod created
  → Scheduler tries to place Pod
  → Node selected
  → Volumes attached
  → Images pulled
  → Containers start   ← Running begins here
```

---

### Two Categories of Pending

| Phase | Meaning |
|---|---|
| **Scheduler Pending** | No suitable node found — `NODE = <none>` |
| **Kubelet / Runtime Pending** | Node selected but startup dependencies blocked (storage, image pull) |

This distinction drives the entire debugging approach.

---

### Common Causes

#### 1. Insufficient Resources

The most common cause. The scheduler uses **requests**, not limits — so even if actual usage is low, the node may be considered full.

```
0/5 nodes available: 5 Insufficient memory
```

```yaml
resources:
  requests:
    memory: 8Gi   # scheduler reserves this, regardless of actual usage
```

#### 2. Node Selector / Affinity Mismatch

```yaml
nodeSelector:
  disktype: ssd
```

If no node has the label `disktype=ssd`, the Pod cannot be placed. Same applies to `nodeAffinity`, `podAffinity`, `podAntiAffinity`, and `topologySpreadConstraints`.

```
0/5 nodes available: 5 node(s) didn't match node selector
```

#### 3. PersistentVolume Problems

Very common with StatefulSets.

```
pod has unbound immediate PersistentVolumeClaims
```

```
Pod
 └─ needs PVC
      └─ PVC waiting for PV   ← blocked here
           └─ Pod stuck Pending
```

Possible causes: no matching `StorageClass`, no available PV, wrong access mode, wrong size, or a broken storage provisioner.

#### 4. Taints and Tolerations

```
node(s) had taint that the pod didn't tolerate
```

A node taint (e.g. `node-role.kubernetes.io/control-plane:NoSchedule`) blocks scheduling unless the Pod has a matching toleration.

#### 5. AZ / Volume Topology Conflict

A particularly common production issue in multi-AZ clusters.

```
EBS volume exists in AZ-a
Pod rescheduled into AZ-b
→ volume node affinity conflict → Pod stuck Pending forever
```

EBS volumes are AZ-bound. Without `WaitForFirstConsumer` binding mode, volumes can be provisioned in the wrong zone before the scheduler has decided where to place the Pod.

#### 6. Cluster Autoscaler Delay

If all nodes are full and the Cluster Autoscaler is configured, the scheduler marks the Pod `Unschedulable`, CA triggers a scale-up, but the Pod stays `Pending` until the new node joins and is ready.

#### 7. Image Pull Issues (Perceived as Pending)

Strictly speaking, this transitions to `ContainerCreating` → `ImagePullBackOff`, but users often perceive it as "stuck Pending." Causes: bad image name, registry auth failure, network issue, or slow pull.

#### 8. PodDisruptionBudget Side Effects

After a drain or failure, a replacement Pod can appear stuck if the old Pod is still in `Terminating` and a PDB prevents the required number of replicas from being satisfied simultaneously.

---

### The Scheduler's Decision Criteria

The scheduler evaluates every node against hard requirements. If **any** fail, the Pod stays `Pending`:

- CPU and memory requests
- Taints and tolerations
- Node / Pod affinity rules
- Storage topology constraints
- Topology spread constraints
- Max Pod density per node
- Resource pressure conditions

---

### Debugging Flow

#### Step 1 — Was a Node Assigned?

```bash
kubectl get pod <pod> -o wide
```

- `NODE = <none>` → scheduler-level problem (capacity, affinity, taints)
- Node assigned → kubelet/runtime/storage-level problem

#### Step 2 — Describe the Pod

```bash
kubectl describe pod <pod>
```

Read the `Events` and `Conditions` sections. Kubernetes almost always states the exact reason here.

#### Step 3 — Check PVCs

```bash
kubectl get pvc
```

If any PVC is in `Pending`, the problem is storage — check the `StorageClass`, provisioner logs, and available PVs.

#### Step 4 — Check Node State

```bash
kubectl describe nodes
```

Look at allocatable resources, taints, and pressure conditions (`MemoryPressure`, `DiskPressure`).

---

### Summary

When I see a Pod stuck in `Pending`, my first move is always `kubectl describe pod` — the Events section usually tells me the exact cause within seconds.

The key mental model is the two-phase distinction: if no node has been assigned, the problem is with the scheduler — I look at capacity, affinity rules, taints, and topology constraints. If a node was assigned but the Pod is still not running, the problem is downstream — storage attachment, image pull, or a runtime dependency.

The cause I see most often in production is resource requests being set too high relative to available node capacity, or EBS volumes being provisioned in the wrong availability zone before the scheduler places the Pod. The AZ issue in particular is a silent trap — the Pod stays `Pending` indefinitely with a `volume node affinity conflict` message that is easy to miss if you are not looking at PVC events.

The important interview point is that `Pending` is almost never an application problem. It is the infrastructure or orchestration layer telling you it cannot fulfil a hard requirement — capacity, storage, placement constraints, or node readiness.

---

## Scenario: Requests, Limits, QoS and the HPA

> **Question:** A Deployment runs one container with `requests.cpu: 100m`, `limits.cpu: 1`, `requests.memory: 128Mi`, `limits.memory: 256Mi`. An HPA targets 70% CPU utilization, `minReplicas: 2`, `maxReplicas: 10`. Under steady load each Pod uses ~250m CPU and ~200Mi memory.
> 1. What QoS class do the Pods get?
> 2. What does the HPA do, and to how many replicas?
> 3. A colleague proposes setting requests equal to limits to get Guaranteed QoS. What changes for scheduling, for the HPA, and for cost?

> **Goal:** Show that one number — `requests` — drives three separate systems: the scheduler, the HPA, and the cloud bill.

### Key Mechanisms Involved

- QoS class assignment (`Guaranteed` / `Burstable` / `BestEffort`)
- Scheduler bin-packing on requests
- HPA utilization formula
- Cluster Autoscaler / Karpenter provisioning on pending requests

---

### Part 1 — QoS Class

**Burstable.** Requests are set but are lower than limits.

`Guaranteed` requires `requests == limits` for **both CPU and memory, in every container** of the Pod (init containers included). Matching CPU alone is not enough.

---

### Part 2 — What the HPA Does

The HPA measures utilization against **requests**, not limits:

```
utilization       = current usage / requests.cpu
                  = 250m / 100m = 250%

desiredReplicas   = ceil(currentReplicas × currentUtilization / targetUtilization)
                  = ceil(2 × 250 / 70) = ceil(7.14) = 8
```

The HPA scales **up to 8**. If total load stays around 500m, each Pod now uses ~62.5m → 62.5% utilization. The next evaluation gives `ceil(8 × 62.5 / 70) = 8`, so it holds at 8.

**Why requests and not limits?** The question the HPA asks is "how full is the capacity reserved for this Pod?" Limits are only a kernel-enforced ceiling, and many containers have no limit at all, so they cannot be the denominator. A container with **no CPU request** cannot be scaled on CPU utilization at all: the HPA reports `missing request for cpu`.

**Common wrong answer:** "250m is 25% of the 1-CPU limit, so the HPA does nothing." That uses the wrong denominator and predicts the opposite of what happens.

---

### Part 3 — Setting Requests = Limits

New spec: `requests.cpu: 1`, `requests.memory: 256Mi` (both raised to match limits).

#### Scheduling

Each Pod now reserves **1 full CPU** instead of 100m, which is 10× more. The scheduler packs on requests, so far fewer Pods fit on each node. Pods can go `Pending` on nodes that have CPU idle in practice but already reserved on paper (see *Pod Stuck in Pending State* above).

#### HPA Behavior

The denominator moves from 100m to 1000m:

```
utilization     = 250m / 1000m = 25%
desiredReplicas = ceil(8 × 25 / 70) → scales down, floored at minReplicas = 2
```

The HPA scales **down** to 2, the opposite of before. This is the "requests too high → HPA under-scales" failure mode. The HPA only reacts once each Pod is near its full core.

#### Cost

Cost **goes up**, even though actual usage is unchanged. The contract of a request is "this capacity is mine whether I use it or not". That is exactly why it costs money: the scheduler and the node autoscaler (Cluster Autoscaler / Karpenter) count **requests**, not usage, when deciding whether nodes are full and whether to add more.

| | Replicas | CPU reserved | CPU actually used |
|---|---|---|---|
| Before (100m request) | 8 | 8 × 100m = **800m** | ~500m |
| After (1 CPU request) | 2 | 2 × 1000m = **2000m** | ~500m |

That is 2.5× more reserved CPU for the same work. The failure domain also gets worse: losing 1 of 2 replicas removes 50% of capacity, while losing 1 of 8 removes 12.5%.

---

### When Guaranteed *Is* the Right Call

- **Dedicated cores.** With the kubelet's `static` CPU Manager policy, Guaranteed Pods with **integer** CPU requests get exclusive cores. This removes noisy-neighbour jitter for latency-sensitive workloads.
- **Eviction order.** Guaranteed Pods are evicted last under node memory pressure.
- **Predictability over efficiency.** Databases and other stateful workloads, where a burst being denied matters less than never being squeezed.

The trade-off is that Guaranteed Pods can never use the idle capacity of the node.

---

### Senior-Level Insight

`requests` is not a "resource hint". It is the single input that the scheduler packs on, the HPA divides by, quota counts against, and the node autoscaler buys hardware for. Changing it to fix one of those (QoS) silently changes the other three. Set requests from observed steady-state usage (p50–p75), let limits absorb bursts, and treat any request change as a scaling and cost change too.

---

### Summary

The Pods are Burstable because requests are below limits. The HPA divides usage by **requests**, so 250m against 100m is 250% utilization, and it scales from 2 to 8 replicas. Raising requests to match limits gets Guaranteed QoS, but it reserves 10× more CPU per Pod, flips the HPA to under-scaling (25% → back to 2 replicas), and raises cost, because nodes are bought for reservations, not usage. Guaranteed is worth it for latency-sensitive workloads that benefit from dedicated cores, not as a default.

---

### What This Scenario Doesn't Cover Yet

- **Memory-based HPA:** why scaling on memory is usually a bad signal (memory rarely drops after scale-out, especially with GC'd runtimes).
- **CPU limits debate:** CFS quota throttling at sub-100ms periods, and why some teams set CPU requests but no CPU limits.
- **HPA tuning:** the 10% tolerance, `behavior.scaleDown.stabilizationWindowSeconds`, and how they interact with the ceil() overshoot seen here.
- **VPA:** automating request right-sizing, and why VPA and HPA conflict on the same CPU/memory metric.

---

## Scenario: `kubectl drain` Blocked by a PodDisruptionBudget

> **Question:** You're rolling a node group to a new AMI. You run `kubectl drain` on the first node, and twenty minutes later it's still hanging, printing over and over:
>
> ```
> error when evicting pods/"payments-7c9f8d-x2k4p" -n payments (will retry after 5s):
> Cannot evict pod as it would violate the pod's disruption budget.
> ```
>
> 1. What is actually happening at the API level? Which API is `drain` calling, and who is refusing it?
> 2. Give two different root causes for this exact symptom: one where the PDB is doing its job, one where it's misconfigured.
> 3. How do you get the rollout moving without causing an outage, and what would you *not* do even though it would unblock you immediately?

> **Goal:** Trace the request chain from `kubectl` to the PDB's status, read the error before acting on it, and choose a fix that keeps the availability guarantee the PDB exists to provide.

### Key Mechanisms Involved

- `kubectl drain` as client-side orchestration (cordon → list → evict → wait)
- The Eviction API (`pods/<name>/eviction` subresource, `policy/v1` `Eviction`)
- PDB selection: label selector → pods, never pod → PDB
- The disruption controller and `status.disruptionsAllowed`
- Optimistic concurrency on the PDB status

Background on why PDBs only gate the Eviction API (and not a Deployment's own rollout deletes) is in `scheduling.md`'s *PodDisruptionBudget: How "Voluntary" Is Actually Checked*. This scenario follows a single eviction through the system.

---

### Part 1 — What `drain` Actually Does

`drain` is **not** a server-side operation. There is no "drain" endpoint. It is client-side logic inside `kubectl` that makes ordinary API calls, in this order:

```
1. PATCH  /api/v1/nodes/worker-3
          {"spec":{"unschedulable":true}}                         ← cordon

2. GET    /api/v1/pods?fieldSelector=spec.nodeName=worker-3       ← find the node's pods

3. filter the list:
     DaemonSet-owned pods  → refuse unless --ignore-daemonsets (then skip them)
     mirror (static) pods  → skip; the API can't remove them
     pods using emptyDir   → refuse unless --delete-emptydir-data
     pods with no owning controller → refuse unless --force

4. for each remaining pod, in parallel:
     POST /api/v1/namespaces/payments/pods/payments-7c9f8d-x2k4p/eviction
          {"apiVersion":"policy/v1","kind":"Eviction",
           "metadata":{"name":"payments-7c9f8d-x2k4p","namespace":"payments"}}

       429 → print "will retry after 5s", sleep, POST again
       404 → pod already gone, fine
       anything else → give up on that pod, drain exits non-zero

5. poll until every evicted pod is gone (or replaced by a new UID)
```

`--timeout` defaults to `0`, which means wait forever. That is why the drain "hangs" for twenty minutes instead of failing: step 4 is stuck in its 429 → sleep → retry loop.

So the API server never sees a "drain". It sees one Eviction request per pod and answers each one on its own. **The thing refusing is the API server's eviction handler**, deciding from the state of the PDB at the moment the request arrives.

---

### Part 2 — Read the Error Before Acting On It

Different eviction failures need different fixes, and the HTTP status tells you which case you're in.

| Response | Message | Meaning | Does retrying help? |
|---|---|---|---|
| `429 Too Many Requests` | `Cannot evict pod as it would violate the pod's disruption budget.` | The pod's single PDB has `disruptionsAllowed: 0` right now | Yes, if pod health recovers. `drain` retries automatically |
| `429 Too Many Requests` | same message, cause `The disruption budget X is still being processed by the server.` | The PDB was just created or edited, and the disruption controller hasn't recomputed its status yet | Yes, within seconds |
| `500 Internal Server Error` | `This pod has more than one PodDisruptionBudget, which the eviction subresource does not support.` | Two PDBs' selectors both match this pod | **No.** It fails every time until the selectors are fixed |
| `403 Forbidden` | `pdb disruptions allowed is negative` | Corrupt or hand-edited PDB status | No |

The 500 case looks like "the drain is blocked by PDBs", but no amount of scaling fixes it, and `drain` doesn't retry it either: it errors out on that pod. The fix is to change one of the PDBs' selectors so each pod is covered by exactly one.

The output in the question is the first 429 case. Before changing anything, confirm it:

```bash
kubectl get pdb -n payments
# NAME       MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# payments   1               N/A               0                     41d

kubectl get pdb payments -n payments -o yaml     # read .status
kubectl get pods -n payments -l app=payments -o wide
```

The `ALLOWED DISRUPTIONS` column is `status.disruptionsAllowed`, the exact number the eviction handler checks (Part 4).

---

### Part 3 — Which Pods a PDB Covers

The relationship goes **only one way**: a PDB declares a label selector, and every pod in the **same namespace** whose labels match is covered by it. The pod spec has no reference to any PDB, and neither does the Deployment. Nothing stops two PDBs from matching the same pod, which leads to the 500 above.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payments
  namespace: payments
spec:
  minAvailable: 1            # or maxUnavailable: N / "N%" (use one, not both)
  selector:
    matchLabels:
      app: payments
  unhealthyPodEvictionPolicy: IfHealthyBudget   # the default (see Part 5)
```

The eviction handler does the matching at request time. It lists the PDBs in the pod's namespace and keeps the ones whose selector matches the pod's labels:

- **0 matches** → the pod has no budget, so it is deleted right away.
- **1 match** → check that PDB's status (Part 4).
- **More than 1** → 500.

Two selector traps:

- **An overly broad selector.** `app: payments` also matches a `payments-migrations` Job's pods, or a second Deployment that reuses the label. Those pods then count toward the same budget.
- **An empty selector.** In `policy/v1`, `selector: {}` matches **every pod in the namespace**. In the old `policy/v1beta1` it matched none, so manifests carried over from the beta API changed meaning.

---

### Part 4 — How the Eviction Is Decided: `status.disruptionsAllowed`

Two components are involved, and they don't talk to each other directly. They communicate through the PDB's `status`.

#### The disruption controller (kube-controller-manager) computes the budget

It watches pods and PDBs and continuously recomputes the status of each PDB:

```
expectedPods        = from the owning controller's scale (Deployment/StatefulSet/RS replicas)
                      when the budget is maxUnavailable or a percentage,
                      otherwise the number of matching pods
currentHealthy      = matching pods that are Ready, excluding pods already being deleted
                      and pods listed in status.disruptedPods
desiredHealthy      = minAvailable                              (minAvailable form)
                    = expectedPods − maxUnavailable             (maxUnavailable form)
disruptionsAllowed  = max(0, currentHealthy − desiredHealthy)
```

It writes these numbers into `status` along with `observedGeneration`. "Healthy" means the pod's `Ready` condition is true, so readiness probes feed straight into this number.

#### The eviction handler (kube-apiserver) spends the budget

For each Eviction request:

```
1. pod is Succeeded / Failed / Pending, or already has a deletionTimestamp
       → delete it, no PDB check (it isn't serving, nothing to protect)
2. find matching PDBs (Part 3): 0 → delete, >1 → 500
3. pod is not Ready → apply unhealthyPodEvictionPolicy (Part 5)
4. status.observedGeneration < metadata.generation → 429 "still being processed"
5. status.disruptionsAllowed == 0 → 429 "needs N healthy pods and has M currently"
6. otherwise:
     disruptionsAllowed -= 1
     add the pod to status.disruptedPods with a timestamp
     UPDATE the PDB status          ← resourceVersion conflict → re-read and retry
7. delete the pod (graceful; it gets a DisruptionTarget condition)
```

Two details in step 6 are what make this safe under parallel evictions:

- **The decrement is an optimistic-concurrency write.** `drain` evicts pods in parallel, and Cluster Autoscaler or Karpenter may be evicting at the same moment. Two requests can't both spend the last unit of budget: the second one's status UPDATE fails on `resourceVersion`, it re-reads the PDB, sees `0`, and gets a 429.
- **`disruptedPods` covers the gap between "eviction accepted" and "pod not Ready".** The evicted pod might still be `Ready` for a moment while it shuts down. Without this list, the controller could recompute a `disruptionsAllowed` that counts the pod as healthy again and hand the same unit out twice. The controller drops entries once the pod is gone, or after about two minutes if the pod was never deleted.

---

### Part 5 — Root Causes

#### The PDB is doing its job

`payments` runs 2 replicas with `minAvailable: 1`. One replica, on a different node, was OOMKilled and is in CrashLoopBackOff, so it isn't Ready:

```
currentHealthy = 1   (only the pod on worker-3)
desiredHealthy = 1
disruptionsAllowed = 0
```

Evicting the pod on worker-3 would leave zero serving pods, so the 429 is correct. The drain is waiting for the app to become healthy again, which is exactly what a PDB is for. The variant that never resolves itself: the other replica isn't crashing but is `Pending`, because there's no room on the other nodes (see *Pod Stuck in Pending State*).

#### The PDB is misconfigured

`payments` runs **1 replica** with `minAvailable: 1`, or `maxUnavailable: 0`, or `minAvailable: 100%`:

```
currentHealthy = 1, desiredHealthy = 1 → disruptionsAllowed = 0, permanently
```

This PDB can never allow a single eviction, however healthy the app is. It blocks every voluntary node operation: drains, autoscaler scale-down, and managed node group upgrades. EKS managed node groups give up with `PodEvictionFailure` after about 15 minutes unless the upgrade is forced. The budget promises availability that one replica can't deliver anyway: when the node dies on its own, the single pod goes with it.

#### The PDB is protecting pods that are already broken

Both replicas are crashlooping. `currentHealthy = 0`, `desiredHealthy = 1`, so the budget is already violated, and under the default `unhealthyPodEvictionPolicy: IfHealthyBudget` an unhealthy pod can only be evicted while the app is *not* disrupted (`currentHealthy >= desiredHealthy`). The PDB now holds the node hostage for pods that serve nothing. `unhealthyPodEvictionPolicy: AlwaysAllow` lets non-Ready pods be evicted whatever the budget says, and is usually the right setting for stateless services.

---

### Part 6 — Unblocking Without an Outage

Fix the cause the error points to:

| Diagnosis | Fix |
|---|---|
| PDB doing its job (a replica is unhealthy) | Fix the unhealthy replica, or add headroom: raise the replica count so `currentHealthy` exceeds `desiredHealthy`. New pods won't land on the cordoned node, and the drain's next retry gets through |
| Replacement pods `Pending` | Add capacity. Cluster Autoscaler or Karpenter react to unschedulable pods by themselves, so there's nothing to "edit" there. Better: **surge first**, bringing up new-AMI nodes *before* draining old ones. EKS managed node group upgrades do this |
| Misconfigured PDB (1 replica) | Decide deliberately: either accept a brief outage of that service now (announce it, relax the PDB, drain), or run 2+ replicas so the budget can mean something. Either way, fix it in the source of truth so the next roll doesn't stall |
| Broken pods held by the PDB | `unhealthyPodEvictionPolicy: AlwaysAllow` |
| 500, overlapping PDBs | Fix one PDB's selector so each pod has exactly one budget |

#### What not to do

- **`kubectl delete pod`** or **`kubectl drain --disable-eviction`**. Both use plain DELETE instead of the eviction subresource, so the PDB is **never consulted**. It unblocks you immediately by removing the safety check for every pod on the node, including pods whose budget was correctly at 0.
- **Silently deleting or loosening the PDB** to make the drain finish. Same result: the guarantee is gone for the window, and nobody decided that was acceptable.
- **Reaching for `--force`.** It does **not** bypass PDBs. It only allows deleting pods that no controller manages (and those pods are then gone for good).

---

### Senior-Level Insight

A blocked drain is the PDB working as designed: it's a guard sitting on the Eviction API, telling you that removing this pod right now would break an availability promise. The useful skill is reading **which** promise. Read the HTTP status first (429 vs 500), then `ALLOWED DISRUPTIONS`, then `currentHealthy` vs `desiredHealthy`. Those three tell you whether the app is unhealthy, the cluster is out of capacity, or the budget itself is wrong. And PDBs only work if every voluntary path uses evictions: one `kubectl delete pod` in a runbook, or a tool that drains with `--disable-eviction`, quietly turns every PDB in the cluster off.

---

### Summary

`kubectl drain` is client-side: it cordons the node, lists the node's pods by `spec.nodeName`, and POSTs an `Eviction` to each pod's `eviction` subresource, retrying every 5s on 429 and, by default, forever. The API server's eviction handler finds the pod's PDB by matching each PDB's label selector against the pod (more than one match → 500, not retryable), then checks `status.disruptionsAllowed`. That number is computed by the disruption controller as `currentHealthy − desiredHealthy` from Ready pods, and spent by the handler through an optimistic-concurrency status update. A 429 means the budget is 0: either a replica is unhealthy (correct), the PDB can never allow a disruption (1 replica with `minAvailable: 1`), or it's guarding pods that are already broken. Fix whichever the numbers show. Never route around the check with a plain pod delete or `--disable-eviction`.

---

### What This Scenario Doesn't Cover Yet

- **Making a manual scale-up stick.** `kubectl scale` in a cluster that also runs an HPA and ArgoCD with self-heal: what reverts it, and where the replica change has to go instead (open question from the discussion).
- **Karpenter and Cluster Autoscaler disruption.** How each one calls the Eviction API when consolidating nodes, Karpenter's `terminationGracePeriod` as a hard deadline on PDB-blocked nodes, and the `do-not-disrupt` / `safe-to-evict` annotations.
- **Graceful shutdown after an accepted eviction.** Endpoint removal racing SIGTERM, and `preStop` sleeps, i.e. why a "successful" drain can still drop requests (see *Replace an Unresponsive Worker Node*).
- **StatefulSets with PDBs.** Quorum-based budgets (e.g. `maxUnavailable: 1` on a 3-node cluster) and how that interacts with `OrderedReady`.

---

## Scenario: QoS and Pod Priority Under Memory Pressure

> **Question:** A worker node has **10Gi memory capacity, 8Gi allocatable**, and runs the kubelet with `--eviction-hard=memory.available<500Mi`. Two PriorityClasses exist: `critical` (value `100000`) and `default` (value `0`). The node runs these five pods (single container each; CPU is set the same way as memory, so it doesn't change any pod's QoS class):
>
> | Pod | PriorityClass | Memory request | Memory limit | Current usage |
> |---|---|---|---|---|
> | **A** `ledger-db` | default | 2Gi | 2Gi | 2Gi |
> | **B** `payments-api` | critical | 1Gi | 4Gi | 3Gi |
> | **C** `reports` | default | 2Gi | 3Gi | 1.5Gi |
> | **D** `fraud-sidecar` | critical | — | — | 500Mi |
> | **E** `batch-export` | default | 512Mi | 2Gi | 1.5Gi |
>
> 1. Give each pod's QoS class.
> 2. Memory climbs slowly until `memory.available` drops below 500Mi. In what order does the kubelet evict, and by what rule? Where does D land relative to "BestEffort goes first"?
> 3. On another day memory spikes too fast for the kubelet, and the **kernel** OOM killer fires. Who dies first, and why can it differ from part 2? Does priority matter?
> 4. A new pod **F** (`critical`, Guaranteed, `requests = limits = 3Gi`) fits on no other node. What happens, who gives way, and which concept decides? Does A's Guaranteed QoS protect it?
> 5. One engineer says "make payments-api Guaranteed so it can't be killed", another says "give it a higher PriorityClass". What does each protect against and what not? What kills B that neither protects against?

> **Goal:** Know which component (scheduler, kubelet, kernel) makes each kill decision, and which input each one reads: QoS, priority, requests, or actual usage.

### Key Mechanisms Involved

- QoS class assignment
- Kubelet node-pressure eviction ranking
- Kernel OOM killer and `oom_score_adj`
- Scheduler preemption
- Container memory limit (cgroup OOM)

### The Map: Who Decides, and What They Read

Four separate mechanisms can kill B. None of them reads both QoS and priority directly:

| Mechanism | Component | Trigger | Reads QoS? | Reads priority? | Reads |
|---|---|---|---|---|---|
| Preemption | scheduler | a higher-priority pod doesn't fit | no | **yes** | requests vs allocatable |
| Node-pressure eviction | kubelet | `memory.available` < threshold | no (indirectly via requests) | **yes** | usage vs requests |
| Kernel OOM kill | Linux kernel | node is out of memory right now | **yes** (via `oom_score_adj`) | no | usage, request, capacity |
| Limit OOM kill | Linux kernel (container cgroup) | container exceeds `limits.memory` | no | no | the container's own usage |

The scenario is these four rows, one per part.

---

### Part 1 — QoS Classes

| Pod | QoS | Why |
|---|---|---|
| A | Guaranteed | requests == limits for CPU and memory |
| B, C, E | Burstable | requests set, below limits |
| D | BestEffort | no requests or limits at all |

---

### Part 2 — Kubelet Node-Pressure Eviction

The kubelet ranks pods for memory eviction by three keys, in order:

1. **Usage above request?** Yes first.
2. **Priority.** Lower first.
3. **Usage − request.** Larger first.

**The kubelet never looks at the QoS class.** QoS shows up only as a side effect of key 1. A BestEffort pod's request is 0, so any usage puts it above request. A Guaranteed pod is capped at its limit, which equals its request, so it can never be above request.

| Group | Pods | Sort | Order |
|---|---|---|---|
| Above request | B (critical, +2Gi), D (critical, +500Mi), E (default, +1Gi) | E has the lowest priority. Then B beats D on excess | **E → B → D** |
| At/below request | A (default, 0), C (default, −0.5Gi) | Same priority; A's excess is larger | **A → C** |

**D, the BestEffort pod, is evicted last of the pods above their request**, because priority (key 2) outranks the size of the excess. "BestEffort first" only holds when priorities are equal.

**The kubelet evicts one pod, then checks again.** Evicting E frees 1.5Gi, well past the 500Mi threshold, so in practice **only E is evicted**. The rest of the order only matters if pressure keeps building. An evicted pod ends up `Failed` with reason `Evicted`, and its controller creates a replacement, possibly on another node.

---

### Part 3 — Kernel OOM Killer

#### Why the kernel gets there first sometimes

The kubelet checks memory periodically. `--eviction-hard` is a buffer meant to give it time to act before the node actually runs out. If memory spikes faster than that loop runs, the kernel can't satisfy an allocation and handles it itself, immediately. The kernel knows nothing about pods, priorities, or the kubelet's ranking. It kills **one process**, and the kubelet sees only the aftermath: the container is reported `OOMKilled` (exit 137) and restarts **in place** under its restart policy. The pod isn't evicted or rescheduled.

#### What the kernel reads

Two numbers per process: how much memory it holds (as a share of the node, 0–1000 points), plus a bias `oom_score_adj` (−1000 to +1000). The process with the highest sum dies. The kubelet sets each container's bias from its QoS class:

| QoS | `oom_score_adj` |
|---|---|
| Guaranteed (and node-critical pods) | −997 |
| BestEffort | 1000 |
| Burstable | `1000 − 1000 × request / capacity`, clamped to 2–999 |

#### The intuition, without memorising numbers

Put the two together for any non-Guaranteed pod and the formula simplifies:

```
score ≈ usage share + bias
      = 1000 × usage/capacity  +  1000 − 1000 × request/capacity
      = 1000 + 1000 × (usage − request) / capacity
```

A BestEffort pod fits the same formula with `request = 0`. So the rule to remember is:

> **The kernel kills whoever is furthest above their request, measured as a share of the node. Guaranteed pods are exempt. Priority is invisible.**

That's the same idea as the kubelet's keys 1 and 3, without key 2. The bias works as a **credit for what you reserved**: a request is a claim on the node, and the kernel won't hold memory inside your claim against you, only memory beyond it. A small request with large usage is what puts a pod at the top of the list, not being BestEffort. BestEffort only means the credit is zero.

Applied (capacity 10Gi):

| Pod | Usage − request | ≈ score |
|---|---|---|
| **B** | +2Gi | **1200**, dies first |
| E | +1Gi | 1100 |
| D | +0.5Gi | 1050 |
| C | −0.5Gi | 950 |
| A | Guaranteed | ~0 (exempt) |

**B, the critical payments-api, is the kernel's first choice**, while the kubelet would have killed it second, after E. Its `critical` priority protected it in part 2 and is worth nothing here.

#### What to take into account for node memory kills

- **The gap between request and usage is the main input.** Both the kubelet and the kernel target pods that use far more than they reserved. Requests set from real usage keep a pod out of the firing line of both.
- **Priority only helps with the kubelet**, never with the kernel. A high-priority pod with an understated request is still exposed.
- **Guaranteed is the only real exemption from both.**
- **A node-level OOM kill means the kubelet's buffer lost the race.** If a container shows `OOMKilled` while well below its own limit, it was a node OOM, not a limit OOM. The node gets a `SystemOOM` event. The fix is node-level: total requests understating real usage, or `--eviction-hard` / `kube-reserved` / `system-reserved` too thin for how fast memory moves on that node.

---

### Part 4 — Preemption

Preemption is a **scheduler** decision, made from **requests against allocatable**. Actual usage doesn't matter, and QoS isn't read at all.

```
requests on node = 2 + 1 + 2 + 0 + 0.5 = 5.5Gi of 8Gi allocatable → 2.5Gi free
F needs            3Gi                                            → 0.5Gi must be freed
```

- **Candidates:** only pods with lower priority than F, i.e. A, C and E. B and D share F's priority and can't be preempted. **A's Guaranteed class gives it no protection**: preemption only looks at priority.
- **Choosing victims:** the scheduler removes all candidates on paper, then puts back as many as still lets F fit, most important first (higher priority, then older pod). The result is the **smallest set**: one pod, either E (freeing exactly 0.5Gi) or C, depending on which is put back last. Never both.
- **Then:** F gets `status.nominatedNodeName`, the victim is deleted gracefully (PDBs respected on a best-effort basis), and F binds once the victim is gone.

**Knock-on effect:** F adds 3Gi of real usage to the node. If that pushes the node into pressure, part 2 runs again. F is now the safest pod there (Guaranteed, top priority), and B, still 2Gi above its request, is next in line.

---

### Part 5 — Guaranteed vs Higher Priority

| Change | Protects B against | Leaves B exposed to |
|---|---|---|
| Make it Guaranteed | Kubelet eviction (it can never be above its request). Kernel OOM killer (bias −997) | Preemption by higher-priority pods |
| Raise its PriorityClass | Preemption. Being early in the kubelet's order among pods above their request | Kernel OOM killer. Kubelet eviction once it's the only pod left above its request |

**Neither protects against B hitting its own limit.** If B grows past `limits.memory: 4Gi`, the kernel kills it inside **its own cgroup**. That needs no node pressure, and QoS, priority, and the kubelet play no part.

**The actual fix** is neither change as proposed. B requests 1Gi and uses 3Gi, and that one gap explains every part of this scenario: it puts B above its request for the kubelet, at the top of the kernel's list, and makes the node look emptier than it is to the scheduler (which is why F fit). Set B's request to its real steady-state usage, keep the high priority against preemption, and treat Guaranteed as a separate choice, worth it if B is latency-sensitive (see *Requests, Limits, QoS and the HPA*).

---

### Senior-Level Insight

"QoS protects against eviction, priority protects against preemption" is the textbook split, and it holds as far as it goes. What you need to add in production is the actual inputs: **the scheduler reads requests and priority, the kubelet reads usage-over-request and then priority, and the kernel reads usage-over-request with a QoS bias and no priority at all.** The common factor across all three is the gap between a pod's request and its real usage. An understated request makes a pod look cheap to the scheduler and expensive to everything that frees memory, so "critical" pods with understated requests are the ones that die in memory incidents.

---

### Summary

A is Guaranteed, B/C/E are Burstable, D is BestEffort. Under node pressure the kubelet ranks on usage-above-request, then priority, then excess: E → B → D, then A → C, and in practice evicts only E. The kernel OOM killer ignores priority and effectively ranks non-Guaranteed pods by `(usage − request) / capacity`, so it kills B first, as an in-place container restart rather than an eviction. F preempts one lower-priority pod (E or C) by requests alone, and A's Guaranteed class doesn't help it. Guaranteed protects against memory kills but not preemption, and priority the reverse. Neither stops a limit OOM, and the real fix is setting B's request to its actual usage.

---

### What This Scenario Doesn't Cover Yet

- **Soft eviction thresholds** (`--eviction-soft` + grace periods) and `--eviction-minimum-reclaim`: how far past the threshold the kubelet goes before it stops.
- **Disk pressure ranking:** the same three-key logic applied to ephemeral storage, plus image and container garbage collection before any pod is evicted.
- **cgroup v2 details:** `memory.high` throttling before `memory.max` kills, and whether a limit OOM kills one process or the whole container.
- **`preemptionPolicy: Never`:** high priority for queue ordering without the right to preempt.
- **`system-node-critical` / `system-cluster-critical`:** why the built-in classes exist and why workloads shouldn't use them.

---

## Scenario: Upgrading a Production EKS Cluster

> **Question:** You own a production EKS cluster on **1.32** and need to get it to **1.33**. It runs ~40 services delivered by ArgoCD, plus cluster add-ons: VPC CNI, CoreDNS, kube-proxy, cert-manager, a policy engine with a validating admission webhook, and Karpenter for nodes.
>
> 1. **Before you touch anything:** what do you check, and why? Include the one property of an EKS control-plane upgrade that makes pre-checks matter far more than for a normal deploy.
> 2. **Order and skew:** in what order do you upgrade control plane, add-ons, and nodes? What version-skew rule tells you how long nodes can stay on 1.32, and which way does skew never work?
> 3. **The surprise after the control plane upgrade:** ArgoCD shows one app as `ComparisonError: no matches for kind "X" in version "Y"`, but the workload is still running and serving. Why are the live objects fine while ArgoCD is broken? Your pre-check ran `kubectl get` on that resource type and saw nothing wrong. Why didn't that catch it, and what would have?
> 4. **The outage:** during the node roll, every new pod creation in the cluster starts failing, across all namespaces, including replacements for drained nodes. The error mentions a webhook timing out. What happened, why did the node roll trigger it, and what do you do now vs. afterwards?

> **Goal:** Show that you treat an irreversible change as a planned operation: knowing what can break, in what order, and what the fallback is, before the first click.

### Key Mechanisms Involved

- One-way control-plane upgrades
- API version removal
- Version-skew policy (kubelet vs API server)
- Add-on and controller compatibility
- Karpenter drift and node replacement
- Admission webhooks and `failurePolicy`

---

### Part 1 — Before Touching Anything

#### The property that shapes everything: there is no rollback

An EKS control-plane upgrade is **one-way**. Once the API server is on 1.33 there is no button to return it to 1.32, and you have no access to etcd to restore it yourself. A bad application deploy can be reverted in Git in a minute. A bad control-plane upgrade has to be fixed *forward*, under pressure. So you have to find the problems beforehand, and the fallback has to be planned beforehand too (see *The Plan* below).

#### The checks, most important first

**1. Removed API versions.** Every few releases Kubernetes stops serving old API versions: `policy/v1beta1` PodDisruptionBudgets went in 1.25, `flowcontrol.apiserver.k8s.io/v1beta3` in 1.32. Any client still sending a removed version breaks the moment the control plane flips: manifests in Git, Helm chart templates, operators creating objects. Because the flip can't be undone, this is the one check that turns directly into an outage you can't back out of. Finding the callers reliably is harder than it looks (Part 3).

**2. Compatibility of everything that talks to the API server.** Two groups:

- **EKS add-ons**, i.e. VPC CNI, CoreDNS, kube-proxy. Each has versions published per Kubernetes release. They have the widest blast radius: a broken CNI means new pods get no network, and a broken CoreDNS means service discovery fails cluster-wide. If the current version doesn't support 1.33, the add-on upgrade becomes part of the plan.
- **Controllers you run yourself**, i.e. Karpenter, cert-manager, the policy engine, ArgoCD. Each publishes its own supported Kubernetes range, and none of that shows up in the EKS console. **Karpenter matters most**: it's the component that will replace every node during the roll, so if it doesn't support 1.33 you lose the ability to provision nodes mid-upgrade.

**3. Version-skew precondition.** EKS upgrades the control plane **one minor version at a time**, and the kubelets must already be close enough to the target version (the rule is in Part 2). Nodes that drifted behind on old AMIs can block the upgrade, or leave you outside the supported skew right after it.

**4. What changes in the release.** Read the Kubernetes changelog's **Urgent Upgrade Notes** and the **EKS release notes** for the target version, and compare them against the cluster's known quirks: things that work today only because of a default that's changing, or an issue the team has accepted that a behaviour change could make worse. **EKS Upgrade Insights** automates part of this: it checks recent audit logs for deprecated API calls and reports kubelet skew and add-on compatibility per target version.

**5. Readiness for the node roll.** Upgrading means replacing every node, so every workload gets evicted at least once:

- PDBs that can actually allow a disruption (no `minAvailable: 1` on a single replica; see *`kubectl drain` Blocked by a PodDisruptionBudget*).
- Single-replica services that will have a brief outage. Decide in advance whether that's acceptable.
- Capacity to **surge**: replacement nodes come up before old ones are drained. `kubectl drain` only evicts pods; it doesn't create nodes. With Karpenter, the surge comes from **drift**: when the `EC2NodeClass` AMI resolves to a 1.33 image, Karpenter marks old nodes as drifted, launches replacements, then evicts through the Eviction API, respecting PDBs and its own disruption budgets.

**6. Infrastructure prerequisites.** EKS creates new control-plane ENIs during the upgrade and needs a few free IPs in the cluster subnets. A subnet that's nearly full of pod IPs (VPC CNI gives each pod a real VPC IP) can fail the upgrade before it starts.

---

### The Plan: Promote the Upgrade, Don't Just Run It

The checks find the risks you know about. The plan is how you limit the damage from the ones you don't.

#### Go through every environment in order

Run the **same upgrade path** (same versions, same add-on bumps, same order) on dev first, then staging, then prod, and let each one soak under real traffic before moving on. That's the closest thing to a rollback an in-place upgrade has: the irreversible step happens first where it doesn't hurt. It only works if the lower environments match prod: same add-ons, same controllers, same admission webhooks, same API usage. An environment missing the policy engine can't catch Part 4.

#### Blue/green clusters for prod

For prod, the strongest option is not to upgrade the cluster at all. Instead you **replace** it:

1. Build a new 1.33 cluster ("green") next to the current 1.32 one ("blue").
2. Deploy the same workloads to it from Git. With GitOps that's mostly registering the new cluster with ArgoCD under the same environment.
3. Shift traffic gradually (weighted DNS or a load balancer in front of both), watching error rates.
4. Keep blue running until green has soaked. **Rolling back means shifting traffic back**, which gives you the rollback the in-place upgrade lacks.

| | In-place upgrade | Blue/green clusters |
|---|---|---|
| Rollback | None; fix forward only | Shift traffic back to the old cluster |
| Cost | No extra infrastructure | Two clusters for the length of the migration |
| Stateful workloads | Volumes stay attached to the cluster | Hard part: EBS volumes are AZ- and cluster-bound, so data has to be migrated or kept outside the cluster (RDS, S3) |
| Prerequisites | Working node-roll hygiene (PDBs, surge) | Everything reproducible from Git, and traffic routing that can span two clusters |
| Side effect | — | Proves the cluster can be rebuilt from scratch, which is also your disaster-recovery test |

The extra cost is worth paying for prod. It buys the one thing an EKS upgrade otherwise lacks, a rollback, and it tests your disaster recovery for free. What makes it hard is **state**: blue/green is easy for stateless services and difficult for anything holding data on in-cluster volumes. That's one more reason to keep databases in managed services rather than in the cluster.

---

### Part 2 — Order and Skew

#### The skew rule: clients may lag, never lead

Since 1.28 the kubelet may be up to **three minor versions older** than kube-apiserver, and **never newer**. So a 1.33 control plane can run 1.32 (or 1.30) nodes, but a 1.33 kubelet must never talk to a 1.32 API server.

The asymmetry follows from which side guarantees compatibility. The API server is the server, and it promises to keep serving older clients: it still understands the requests, fields and API versions that 1.30–1.32 kubelets send. It can make that promise only for versions that already exist. A newer kubelet may use a field or behaviour that the old API server has never heard of, and that fails in one of two ways:

- **An error.** The request names something the API server doesn't serve, and it's rejected.
- **Silent data loss.** The request carries a field the API server doesn't know, so the API server **drops the unknown field** when it stores the object. The kubelet believes it reported something; the stored object doesn't have it. Nothing alerts on this, which makes it worse than the error.

The one-way skew is what makes an in-place upgrade possible at all: the control plane moves first, and the nodes keep running their workloads on the old kubelet until they're replaced. The API server's promise to older clients isn't open-ended, though. It holds only as long as the API versions those clients use are still served, which is exactly where Part 3 goes wrong.

#### Controllers must span both versions

Your own controllers (Karpenter, cert-manager, the policy engine, ArgoCD) are API clients too, but they don't follow the control plane the way add-ons do. Each one has to work on **both** versions during the transition: on 1.32 before the flip and on 1.33 after it.

- If the current version already supports 1.33, nothing to do.
- If it doesn't, but a newer version supports both 1.32 and 1.33, upgrade the controller **before** the control plane. Then the flip changes nothing from its point of view.
- If no single version covers both, it's a two-step change (controller, control plane, controller again), and the plan has to say so explicitly.

A version that *requires* 1.33 can't go first; it wouldn't run on the 1.32 cluster.

When a controller is incompatible, what breaks is **that controller's reconcile loop**, not the cluster: Karpenter stops provisioning, ArgoCD stops diffing, cert-manager stops renewing. Workloads that are already running keep running. Karpenter is the one that matters most here, because it's what replaces every node in step 4. If it breaks after the flip, the upgrade stalls halfway with no way to provision nodes.

#### Add-ons follow the control plane

The EKS add-ons work the other way round. VPC CNI, CoreDNS and kube-proxy have versions published per Kubernetes release, and CoreDNS and kube-proxy are bumped **after** the control plane, to the versions published for 1.33. kube-proxy in particular must never be newer than kube-apiserver, the same rule as the kubelet. Upgrading it before the control plane breaks skew.

#### The order

| Step | What | Why here |
|---|---|---|
| 1 | Own controllers (Karpenter, cert-manager, policy engine, ArgoCD) to versions supporting both 1.32 and 1.33 | They must survive the flip; Karpenter is the node-roll engine |
| 2 | Control plane 1.32 → 1.33 | The irreversible step; everything after depends on it |
| 3 | EKS add-ons (VPC CNI, CoreDNS, kube-proxy) to their 1.33 versions | They follow the control-plane version |
| 4 | Nodes, via Karpenter drift to the 1.33 AMI | Kubelet last; skew allows it to lag until you get there |

Because EKS upgrades the control plane one minor at a time, going further (say 1.33 to 1.35) repeats steps 1–3 per minor. Nodes can stay behind for up to three minors, but they shouldn't: every skipped roll is a bigger jump later.

---

### Part 3 — The Surprise After the Flip

The control plane is on 1.33. One ArgoCD app goes red:

```
ComparisonError: no matches for kind "HorizontalPodAutoscaler" in version "autoscaling/v2beta2"
```

The HPA is still there, still scaling its Deployment, and `kubectl get hpa payments -o yaml` returns it without complaint.

#### An object isn't stored "as" the version you sent

A resource can be **served** in several versions (`autoscaling/v2beta2`, `autoscaling/v2`), but it's **stored** in etcd in exactly one: the storage version. Every read and write is converted between the version the client asked for and the storage version. The version in your manifest decides how you *talk* about the object, not how it's kept.

A version is never removed while it's still the storage version:

- **Built-in types:** the storage version moves to the new version at least one release before the old one stops being served. Each object is re-encoded in the new version the next time it's written.
- **CRDs:** the same thing is explicit. Each entry in `spec.versions` has `served` and `storage` flags, and `status.storedVersions` records every version that may still be in etcd. The API server refuses to drop a version from the CRD while it's listed there. To remove it, you first run a storage migration (re-write every object), then prune the list.

So removing `v2beta2` only stops the API server **accepting requests that name it**. The objects in etcd are untouched, and anything asking for `v2` gets them as before.

#### Why the workload is fine

Nothing that *runs* the HPA uses `v2beta2`. The HPA controller lives in kube-controller-manager, which ships with the control plane, so after the flip it talks to the API server in whatever 1.33 serves. Scaling continues.

#### Why ArgoCD breaks: it never gets as far as a request

To fetch a live object, a client has to turn the manifest's **group/version/kind** into a URL path (group/version/**resource**). It does that with the **RESTMapper**, a lookup table built from the API server's **discovery** data (the list of groups, versions and resources it currently serves):

1. ArgoCD renders the app from Git and finds `apiVersion: autoscaling/v2beta2`, `kind: HorizontalPodAutoscaler`.
2. It asks the RESTMapper for the matching resource, to build `/apis/autoscaling/v2beta2/namespaces/payments/horizontalpodautoscalers/payments`.
3. Discovery on 1.33 no longer lists `autoscaling/v2beta2`. The lookup fails **inside ArgoCD**: `no matches for kind ... in version ...`.

No HTTP request is ever sent, so ArgoCD can't fetch the live object to diff against. That's why it's a `ComparisonError` and not a sync failure. Only the **desired state in Git** is broken, not the cluster.

The fix is in Git: change the manifest to `autoscaling/v2`. Check the schema too, since a version bump can rename or restructure fields.

#### Why the pre-check missed it

`kubectl get hpa` goes through discovery too, but it doesn't pick a version from a manifest: it asks for the **preferred** version the API server advertises. On 1.32 that already is `v2`, so the output looks the same whichever version created the object, and it would look the same on 1.33. A `kubectl get` on live objects tells you nothing about which versions **clients send**. The problem isn't in the objects; it's in the callers.

#### What would have caught it

Look at the callers, not the objects:

- **Static, from Git:** render every app the way ArgoCD will on the target version, then check the result against what 1.33 serves (`pluto`/`kubent` do this; a server dry-run on a 1.33 cluster does it too). Rendering has to be told the target: charts often choose an `apiVersion` with `.Capabilities.APIVersions.Has "autoscaling/v2"`, and plain `helm template` uses default capabilities. ArgoCD passes the real cluster's API versions to Helm, so a chart written that way switches to `v2` on its own after the flip. Only a hard-coded old version breaks.
- **At runtime, from the API server:** Git doesn't hold every caller. Operators create objects in code, CI and cron scripts call the API directly, and Helm keeps old releases with their old `apiVersion`s. The API server sees all of them. It marks deprecated-API requests in the **audit log** and counts them in the `apiserver_requested_deprecated_apis` metric. This is what **EKS Upgrade Insights** reads (Part 1, check 4).
- **The environment order** from *The Plan*: if dev is managed by ArgoCD from the same Git and is upgraded first, this exact `ComparisonError` shows up there first, where it costs nothing.

---

### Part 4 — The Webhook Outage During the Node Roll

Twenty minutes into the Karpenter roll, every new pod creation fails, in every namespace. The ReplicaSets show:

```
Error creating: Internal error occurred: failed calling webhook "validate.policy.example.io":
failed to call webhook: Post "https://policy-webhook.policy-system.svc:443/validate?timeout=10s":
context deadline exceeded
```

Running pods are fine. But each drained node takes capacity with it, and nothing replaces it.

#### Where the webhook sits in a request

Every write to the API server goes through the same chain before it reaches etcd:

1. Authentication, then authorization (RBAC).
2. **Mutating admission**, including mutating webhooks.
3. Schema validation of the (possibly mutated) object.
4. **Validating admission**, including validating webhooks.
5. Persist to etcd.

For each `ValidatingWebhookConfiguration` whose `rules` match the request (here: `CREATE` on `pods`, all namespaces), the API server POSTs an `AdmissionReview` to the webhook's Service and waits up to `timeoutSeconds` (default 10s). If the call fails (connection refused, no endpoints, timeout), **`failurePolicy` decides the outcome**: with `Ignore` the request carries on as if the webhook had said yes, and with `Fail` it is rejected. Policy engines ship with `Fail`, because a policy that's skipped whenever its backend is down isn't a policy.

The webhook's backend is an ordinary Deployment behind an ordinary Service. When the roll evicted its last pod, the Service had no endpoints, every call failed, and `Fail` turned that into a rejection of every pod create in the cluster.

#### Why it doesn't heal itself: the deadlock

When the webhook pods were evicted, the ReplicaSet controller immediately tried to create replacements. Those creates are pod `CREATE`s too, so they go through the same webhook, which has no backend, so they're rejected. **The webhook's own replacements need the webhook's approval to exist.** Nothing in the cluster will ever break that loop; a human has to.

The rejection happens at admission, before anything is written to etcd. So no Pod object exists: nothing is `Pending`, the scheduler never sees a pod, and `kubectl get pods` shows only what's still running. The evidence is the `FailedCreate` events on the ReplicaSets and the API server's webhook error metrics. Someone who looks for Pending pods in this incident finds nothing.

#### Why the node roll triggered it

The webhook had worked for months, and nothing about it changed in the upgrade. What changed is that **every node was being replaced**, and the webhook Deployment couldn't survive that:

- **Too few replicas, or all on the same node.** One replica, or two that landed on the same node, go away with one drain.
- **No PDB, or a PDB that couldn't hold.** Karpenter drift evicts through the Eviction API and respects PDBs. With no PDB, nothing stops it from taking the last pod.
- **The lower environments didn't catch it**, because they either didn't run the policy engine or had too few nodes to roll for it to matter (*The Plan*: an environment missing the policy engine can't catch Part 4).

#### Now: break the loop

The aim is to let pods be created again, and to keep the policy disabled for as short a time as possible.

1. **Confirm it.** The webhook Service has no endpoints (`kubectl get endpointslices -n policy-system`), and the Deployment has fewer ready replicas than desired, with `FailedCreate` events.
2. **Stop GitOps from undoing the fix.** The `ValidatingWebhookConfiguration` is delivered by ArgoCD. With `selfHeal` on, ArgoCD reverts a manual patch within minutes and the outage comes back. Disable auto-sync on that app first.
3. **Exempt the webhook's own namespace, or failing that, set `failurePolicy: Ignore`.** Adding a `namespaceSelector` that excludes `policy-system` breaks the deadlock and keeps the policy enforced everywhere else. Under time pressure, flipping to `Ignore` is the blunter alternative that always works. Deleting the configuration also works, but then you have to restore it from Git exactly.
4. **Watch the webhook pods come back**, then restore `Fail` (or re-enable sync, so ArgoCD restores it).

`Ignore` costs you something: while it's in place **the policy is not enforced**, so any pod created in that window was admitted without the check. Afterwards, list what was created in the window and re-check it against the policy (most policy engines have a background/audit scan for this).

#### Afterwards: make it impossible, then make it unlikely

**Make the deadlock impossible.** Exclude the webhook's own namespace in the configuration, so it can always bring its own pods back:

```yaml
webhooks:
  - name: validate.policy.example.io
    failurePolicy: Fail
    namespaceSelector:
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values: [policy-system, kube-system]
```

`kube-system` belongs in that list for the same reason. CoreDNS, the VPC CNI and kube-proxy are what every other pod needs in order to work, and the webhook's own pods need them too. A cluster-critical component must never depend on itself, or on the policy engine, to come back. Most policy engines exclude these namespaces by default; check that the default is still there.

**Make losing every replica unlikely.**

- **At least 3 replicas, with a PDB of `maxUnavailable: 1`.** A PDB of `minAvailable: 2` on 2 replicas leaves `disruptionsAllowed` at 0. Karpenter drift then can't evict either pod, and the roll stalls on those two nodes (see *`kubectl drain` Blocked by a PodDisruptionBudget*). The PDB has to allow progress, not just protect.
- **Spread across nodes and zones.** `whenUnsatisfiable: ScheduleAnyway` is the right default for most services. For a component that can block every pod create in the cluster, use `DoNotSchedule` on the hostname key (or required pod anti-affinity), so two replicas never share a node. Keep zone spreading soft, so one AZ outage can't leave the pods unschedulable.
- **A high priority class** (`system-cluster-critical`). The *QoS and Pod Priority* scenario says workloads shouldn't use it. A cluster-wide admission webhook is the kind of component it exists for: it should preempt other pods rather than wait for capacity.
- **A short `timeoutSeconds`** (2–5s instead of 10). Even when the webhook works, every pod create waits on it. When it's down, a long timeout makes every client wait 10s before failing.
- **Alert on the webhook, not just the pods.** The API server exports webhook latency and rejection metrics per webhook (`apiserver_admission_webhook_*`). An alert on rejections caused by call failures fires within a minute; one on a policy-system Deployment with fewer than 3 ready replicas fires before the outage.

---

### Senior-Level Insight

Parts 3 and 4 look like the same lesson twice: something you didn't upgrade breaks during the upgrade. The ArgoCD error came from callers still using a removed API. The webhook outage came from a dependency the change exercised for the first time: every node replaced, in one go. An upgrade is the one routine event that touches **everything at once**, so it finds what quietly couldn't survive a full roll. The useful question in planning isn't only "what changes in 1.33" but "what runs in this cluster that has never lived through every node being replaced". Admission webhooks with `Fail` are first on that list, because they turn an outage of a single small component into an outage of the whole cluster's ability to create pods.

---

### Summary

An EKS control-plane upgrade is one-way, so the risk is found and contained before the flip: removed APIs, controller and add-on compatibility, skew, node-roll readiness, and an environment order (or blue/green clusters) that acts as the missing rollback. The order is controllers first, then control plane, add-ons, and nodes last. After the flip, a removed API version breaks callers (ArgoCD's RESTMapper) and not stored objects, which is why a `kubectl get` pre-check misses it and audit logs or rendered manifests catch it. During the roll, a `Fail` validating webhook whose pods were all evicted rejects every pod create, including its own replacements. Break the loop by exempting its namespace or setting `Ignore` (with ArgoCD's self-heal paused), then make it impossible with a `namespaceSelector` and unlikely with replicas, a PDB that allows progress, hard spreading across nodes, and alerting on webhook failures.

---

### What This Scenario Doesn't Cover Yet

- **Network reachability of webhooks.** The API server calls webhooks from the EKS control-plane ENIs. New nodes with a different security group (a changed `EC2NodeClass` selector) can make a webhook running on them unreachable even with healthy pods.
- **How kube-proxy is upgraded.** It's a DaemonSet, so bumping the add-on puts the new version on every node, including 1.32 nodes not yet rolled. That's within the upstream kube-proxy ↔ kubelet skew allowance (±3 minors since 1.28; it had to match the kubelet's minor before). Not discussed yet.

---

## Scenario: Healthy Pods, Intermittent 504s

> **Question:** `kubectl get pods` shows every pod of the `checkout` service `Running` and `1/1 Ready`, with no restarts. Users intermittently get **504 Gateway Timeout**. The traffic path is client → AWS ALB → ingress-nginx → Service → pods.
>
> 1. What does a 504 tell you before you look at anything, and who in the path generated it?
> 2. Walk through the troubleshooting flow: what you check, in what order, and why.
> 3. Name plausible root causes in different layers of the path.

> **Goal:** Show that you read the status code and the logs to locate the failing hop, instead of guessing layer by layer, and that you use "intermittent" to narrow the cause.

### Key Mechanisms Involved

- What 502, 503 and 504 each mean at a proxy
- ALB and nginx access logs (who generated the code)
- Timeouts along the chain
- How ingress-nginx picks upstreams (pod IPs from EndpointSlices)
- What readiness does and doesn't prove

---

### Part 1 — What the Code Says

A 504 is generated by a **proxy**, not by the app: *"I forwarded the request upstream and got no response before my timeout."* In this path there are two proxies, the ALB and nginx, and either can be the one that gave up. The neighbouring codes mean different things, and telling them apart narrows the cause before you open a dashboard:

| Code | What the proxy is saying | Typical cause |
|---|---|---|
| 502 | Upstream gave a bad answer: connection refused or reset, invalid response | Pod shutting down, app crashed mid-request, keepalive mismatch |
| 503 | There is no upstream to send to | Service has no endpoints, every target unhealthy |
| 504 | Upstream didn't answer in time | Slow app or dependency, or packets dropped without a reply |

So a Service selector that doesn't match the new pods is a **503** (and fails every request), and traffic to a pod that just exited is usually a **502** (connection refused). A 504 means something accepted the work, or swallowed the packets, and then went quiet.

### Part 2 — Find the Hop That Gave Up

Start at the top, because a 504 from deep in the chain can be passed up unchanged. Use the logs to find the hop, rather than inspecting each layer by hand:

1. **ALB access logs** record both `elb_status_code` and `target_status_code` for every request:
   - `504` / `-`: the **ALB** timed out waiting for nginx. Look at the ALB's idle timeout, at nginx being overloaded, and at the network between them (security groups, NACLs on some subnets).
   - `504` / `504`: nginx generated it and the ALB passed it on. Go one hop down.
2. **nginx access logs** have `upstream_addr`, `upstream_status` and `upstream_response_time` per request. Failing requests that all take almost exactly the configured timeout (nginx's `proxy_read_timeout` defaults to 60s, as does the ALB idle timeout) are hitting that limit, not failing quickly.
3. **The app and its dependencies.** Traces or app metrics for the failing requests: is the time spent in the handler, in a database query, or in a call to another service?

Timeouts along the chain should get shorter as you go down: the ALB waits longest, then nginx, then the app's own calls to its dependencies. Then the innermost component fails first, with a clear error, instead of the outer proxy giving up and returning a 504 with no detail.

### Part 3 — Root Causes, by Layer

**The app or what it depends on.** A slow database query, a slow downstream service, or a pool (DB connections, worker threads) that's full, so requests queue. The fix might be config, not code: an app **CPU-throttled** at its limit is slow without being broken (see *Requests, Limits, QoS and the HPA*).

**The network between nginx and the pod.** A timeout doesn't need a slow app, only no reply. Dropped packets make nginx wait for `proxy_connect_timeout`, and the app never sees the request:
- a NetworkPolicy or a security group for pods that covers only some pods
- a node with a broken CNI, or a full conntrack table under load

**Stale endpoints.** ingress-nginx doesn't route through the Service's ClusterIP. It reads EndpointSlices and connects to **pod IPs directly**. During a rollout or a drain, a pod can be gone while its IP is still in nginx's upstream list. If the node is still up, the connection is usually refused (502). If the node is gone, nothing answers (504). This propagation delay is why a `preStop` sleep is needed (*Replace an Unresponsive Worker Node*).

**The proxies themselves.** nginx out of `worker_connections` or CPU-throttled, or a NACL blocking the ALB from the subnets some nginx pods are in. These show up at the ALB as `target_status_code: -`.

#### Why "Ready" proves so little

Readiness usually checks a cheap `/healthz` that never touches the database or other dependencies. A pod can pass it while every real request waits on a dependency. `Ready` means "the process answers its health endpoint", not "requests succeed".

### Part 4 — Use "Intermittent"

Almost every cause above affects **a subset** of traffic: one pod, one node, one subnet or AZ, one rollout window. A cause that affected every request would give 100% errors. So once you've found the hop, group the failing requests:

- by `upstream_addr` in the nginx log: is it one pod IP?
- by node or AZ: do the bad pod IPs share a node, or the subnet a NACL covers?
- by time: does it line up with a deploy, a scale-up or scale-down, or a node drain (stale endpoints)?
- by route: is it one endpoint that does a slow query?

---

### Senior-Level Insight

The junior answer to "pods are healthy but users get 504s" is to walk every layer and check it. The senior answer reads the evidence that already exists: the status code says what kind of failure it is, the ALB's two status columns say which proxy gave up, and the response time says whether it was a timeout. Within a couple of minutes you know where in the chain to look. "Intermittent" then narrows it further, because a fault that affects only some requests is almost always tied to one pod, node, subnet or moment, and that shows up as soon as you group the failing requests.

---

### Summary

A 504 is a proxy (ALB or nginx) saying the upstream didn't answer in time; 502 means a bad answer, and 503 means no upstream at all. ALB access logs show which proxy generated it (`target_status_code` of `-` means the ALB itself timed out), and nginx's `upstream_addr` and `upstream_response_time` show which pod and how long. Causes span the app and its dependencies, silent packet drops between nginx and pods, stale pod IPs in nginx's upstream list during rollouts, and the proxies' own limits. Readiness doesn't rule out any of them. Because the errors are intermittent, group the failing requests by pod, node, subnet and time to find what they share.

---

### What This Scenario Doesn't Cover Yet

- **Keepalive mismatches:** a proxy reusing an idle connection that the app has just closed, which is the classic cause of intermittent **502**s.
- **Retries:** whether nginx or the ALB retries a timed-out request on another pod, and why retrying non-idempotent requests (a checkout) is dangerous.
- **Slow pod startup vs readiness:** a pod marked Ready before its caches or connection pools are warm, giving slow first requests after every scale-up.
