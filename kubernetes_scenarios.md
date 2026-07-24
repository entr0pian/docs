# Kubernetes Scenarios

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
