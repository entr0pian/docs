# Kubernetes Persistent Volumes (PV) & Persistent Volume Claims (PVC)

---

## Core Concepts

### PersistentVolume (PV)

A **cluster-scoped** resource that represents actual storage. It exists independently of any namespace and is managed by cluster administrators.

Can be provisioned in two ways:

| Method | Description |
|---|---|
| **Static** | Admin manually creates PV resources |
| **Dynamic** | Automatically created via a `StorageClass` |

Common storage backends: AWS EBS, AWS EFS, GCP Persistent Disk, NFS, local disk.

---

### PersistentVolumeClaim (PVC)

A **namespace-scoped** request for storage made by a workload or user. Think of it as a ticket asking for storage that meets certain requirements.

A PVC defines:
- Requested **size**
- Required **access modes**
- Target **StorageClass**

> **Key Idea:** PVC = *request* for storage. PV = *actual* storage resource.

---

## Binding Model (1:1 Relationship)

A PV and a PVC have a strict one-to-one relationship — once bound, neither can be used by anything else.

Binding is enforced bi-directionally:

```
PV.spec.claimRef     → points to PVC (name, namespace, UID)
PVC.spec.volumeName  → points to PV (name)
```

The **UID** on the `claimRef` is critical — it prevents rebinding confusion if a PVC is deleted and recreated with the same name.

---

## Binding Process (Controller Behavior)

Managed by the **PV controller** inside `kube-controller-manager`.

```
PVC created
    │
    ▼
Status: Pending
    │
    ▼
Controller searches for matching PV:
  ✓ capacity ≥ requested
  ✓ accessModes match
  ✓ storageClass matches
  ✓ PV not already bound
    │
    ▼
Controller sets:
  PV.spec.claimRef  = this PVC
  PVC.spec.volumeName = this PV
    │
    ▼
Both → Status: Bound
```

---

## Provisioning Types

### Dynamic Provisioning

Triggered automatically when a PVC references a `StorageClass`. A new PV is created on-demand.

**Pros:**
- Exact size allocation — no wasted capacity
- Fully automated — no admin intervention
- Default in most cloud environments

### Static Provisioning

An admin pre-creates PVs, and Kubernetes selects a matching one when a PVC is submitted.

**Cons:**
- PV may be larger than requested (capacity is wasted)
- No splitting or resizing
- Requires careful upfront planning

---

## Reclaim Policies

Defined on the PV (or inherited from the StorageClass). Determines what happens to the PV when its bound PVC is deleted.

| Policy | Behavior | When to Use |
|---|---|---|
| **Delete** | PV and underlying storage are deleted | Cloud volumes (EBS, GCE PD) — default for dynamic |
| **Retain** | PV is kept, data preserved, enters `Released` state | When you need manual control or data recovery |
| **Recycle** | *(Deprecated)* Wiped and made available again | Do not use |

---

## PV Lifecycle States

```
Available → Bound → Released → Available (after manual unlock) → Bound
```

| State | Meaning |
|---|---|
| **Available** | Ready to be claimed |
| **Bound** | Locked to a specific PVC |
| **Released** | PVC deleted, but `claimRef` still set — not reusable yet |

---

## Reusing a PV with `reclaimPolicy: Retain`

When a PVC is deleted, the PV enters `Released` state. The `claimRef` acts as a lock — no new PVC can bind to it until manually cleared.

### Step-by-step procedure

**Step 1 — Verify the PV state**

```bash
kubectl get pv
```

Expected: `STATUS = Released`

---

**Step 2 — Inspect the data**

Before reusing, access the underlying storage and verify:
- Data is valid and expected
- Decide whether to reuse as-is or clean it

> Skipping this step risks data leaks between workloads.

---

**Step 3 — Remove the `claimRef` to unlock the PV**

```bash
# Recommended: JSON patch
kubectl patch pv <pv-name> --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'

# Alternative: set to null
kubectl patch pv <pv-name> -p '{"spec":{"claimRef": null}}'
```

Result: PV status moves from `Released` → `Available`

---

**Step 4 — Create a new PVC**

**Option A — Automatic binding** (Kubernetes picks a matching PV)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: my-sc
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Risk: Kubernetes might bind to a *different* PV that also matches.

**Option B — Explicit binding (recommended)**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: my-sc
  volumeName: <exact-pv-name>      # forces binding to this specific PV
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Using `spec.volumeName` guarantees binding to the exact PV — no ambiguity.

---

**Step 5 — Verify binding**

```bash
kubectl get pv,pvc
```

Expected: both show `STATUS = Bound`, pointing at each other.

---

### Binding Requirements

For a PVC to bind to a specific PV, all of the following must hold:

| Requirement | Detail |
|---|---|
| `storageClassName` | Must match exactly |
| `accessModes` | PVC modes must be a subset of PV modes |
| `storage` | PVC requested size ≤ PV capacity |

---

### Warnings

- Data is **never cleaned automatically** with `Retain`
- Risks of skipping data inspection: stale data, corruption, security leaks
- `Retain` exists to force **operator awareness** — accidental data reuse is prevented by design

---

## Pod ↔ PVC ↔ PV Dependency and Pending States

### Dependency Chain

```
Pod → uses → PVC → binds to → PV
```

Every layer must be ready before the layer above it can proceed:

| Layer | Requires |
|---|---|
| PV | To exist and be `Available` |
| PVC | A matching PV → must reach `Bound` |
| Pod | PVC to be `Bound` → only then can the volume be mounted |

> A pod cannot run unless storage is fully resolved all the way down the chain.

---

### Pod Stays in Pending

A pod referencing a PVC will remain in `Pending` if:

- The PVC is still `Pending` (no matching PV found yet)
- The PVC is `Terminating` (being deleted, finalizer not yet removed)
- No PV exists that satisfies the PVC's requirements

```
Pod created
    │
    ▼
References PVC → PVC is Pending
    │
    ▼
Volume cannot be mounted
    │
    ▼
Pod stays Pending (Unschedulable)
```

To diagnose:

```bash
kubectl describe pod <pod-name>
# Look for: "pod has unbound immediate PersistentVolumeClaims"

kubectl get pvc
# Look for STATUS = Pending
```

---

### Scheduler and Volume Binding

`volumeBindingMode` is a field on the **StorageClass**, and it has exactly two possible values. It controls *when*, relative to pod scheduling, a PVC gets bound (and, for dynamic provisioning, when the underlying volume gets created).

| Binding Mode | PV Provisioned When | Use Case |
|---|---|---|
| `Immediate` | As soon as the PVC is created — before any pod exists | Storage with no topology constraints (e.g. NFS) |
| `WaitForFirstConsumer` | After a pod referencing the PVC is scheduled to a node | Zone-pinned cloud block storage (EBS, GCE PD), local volumes |

#### The `Immediate` pitfall: zone affinity conflict

Cloud block storage (EBS, GCE PD) is created in a single zone and cannot move. With `Immediate`, the volume is provisioned the moment the PVC is created — the CSI provisioner has no idea which node the pod will eventually land on, so it picks a zone with no scheduling context at all.

If the scheduler then needs to place the pod on a node in a *different* zone (due to taints, affinity, capacity, or simple bad luck), it can't — the disk isn't there. The pod sticks in `Pending` with:

```
0/6 nodes are available: 6 node(s) had volume node affinity conflict.
```

This isn't self-healing. The PV's `nodeAffinity` (set to the zone it was created in) never changes, so the fix is to delete the PVC (and, depending on `reclaimPolicy`, the PV) and let it re-provision — there's no way to move an already-created EBS volume to another zone via Kubernetes.

#### `WaitForFirstConsumer` closes the gap

Binding — and for dynamic provisioning, volume creation itself — is deferred until a pod that references the PVC is scheduled. The scheduler picks the node *first*, using its normal constraints, and only then does provisioning happen, scoped to that node's zone. This is implemented as a distinct field on the PVC object, not just a delay: a `volume.kubernetes.io/selected-node` **annotation**, written by the scheduler's `VolumeBinding` plugin once it tentatively reserves a node for the pod. See the walkthrough below for the exact sequence.

Before any pod exists to trigger this, a PVC in this mode sits in `Pending` with a condition message that reflects intentional inaction, not failure:

```
waiting for first consumer to be created before binding
```

Contrast that with the `Immediate` failure message above — that one comes from the **scheduler**, evaluating a pod against a PVC that should already be bound. This one comes from the **PV controller**, evaluating a PVC on its own, before any pod has entered the picture.

> Scheduling and volume binding are interconnected — a wrong binding mode for zone-pinned or local storage can cause pods to be scheduled on nodes where the volume doesn't or can't exist.

---

## Access Modes: `ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany`, `ReadWriteOncePod`

`accessModes` is a field on both PVC (requested) and PV (offered) — the "Binding Requirements" table above already notes that a PVC's requested modes must be a subset of the PV's. What that table doesn't say is that **each mode is a capability contract enforced by a different actor, at a different point in the pipeline** — not a hint passed to the CSI driver about what kind of volume to create (that's `parameters`, e.g. `type: gp3`).

### The four modes

| Mode | Common backing example | What it actually guarantees |
|---|---|---|
| `ReadWriteOnce` (RWO) | EBS, GCE PD — any single-attach block device | Read-write, but the backend can physically only attach to one **node** at a time |
| `ReadOnlyMany` (ROX) | EFS, NFS-style backends | Read-only, concurrently, from any number of nodes |
| `ReadWriteMany` (RWX) | EFS, NFS-style backends | Read-write, concurrently, from any number of nodes — requires a backend that supports true multi-writer semantics at the protocol level |
| `ReadWriteOncePod` (RWOP) | Same backends as RWO | Read-write, but restricted to exactly one **pod** cluster-wide — closes a gap RWO leaves open |

### Enforcement: which controller, at what point

| Mode | Enforced by | Point in the pipeline | Mechanism |
|---|---|---|---|
| RWO | Attach/detach controller (`kube-controller-manager`) + CSI `external-attacher` sidecar | Node-level only, via `VolumeAttachment` objects | Refuses to attach the volume to a second node while already attached elsewhere |
| ROX | kubelet | Local `mount` call on the node | Passes the `ro` flag at mount time — backend permissions and the volume itself are untouched, this is purely local |
| RWX | Nobody in Kubernetes — the backend protocol itself | `CreateVolume`/`ControllerPublishVolume`, at the CSI driver | The driver reports (or refuses) the `MULTI_NODE_MULTI_WRITER` capability; EBS's driver rejects it outright, EFS's driver supports it because NFS does |
| RWOP | **kube-apiserver admission** | Pod creation — before the pod ever reaches the scheduler | Rejects a new pod referencing a PVC that's already in use by another pod, cluster-wide |

RWOP is the odd one out: it's the only mode enforced by the API server itself rather than by a controller loop or a CSI sidecar — and it has to work that way, because the gap it closes exists *before* scheduling and attachment ever happen.

### Common Pitfall: A `Deployment` Sharing an RWO PVC

`Deployment` has no `volumeClaimTemplates` — that's `StatefulSet`-only (see `workloads.md`'s "Storage identity" section). So a `Deployment` with `replicas: 2` that wants persistent storage at all can only do it by hardcoding the *same* PVC name into every replica's pod template. That single decision creates two very different failure modes depending on where the scheduler happens to put the second replica.

**Scenario A — replicas land on different nodes (the common case, fails loudly):**

```
kubectl describe pod <replica-2>
# Warning  FailedAttachVolume
Multi-Attach error for volume "pvc-1234" Volume is already exclusively
attached to one node and can't be attached to another
```

The attach/detach controller is doing exactly its job here — it's already attached the volume to replica 1's node and won't attach it to a second one. Replica 2 sits in `ContainerCreating` indefinitely. Annoying, but safe: no corruption, and the cause is obvious from `kubectl describe`.

**Scenario B — replicas land on the same node (the dangerous case, fails silently):**

RWO's exclusivity guarantee is node-scoped, not pod-scoped — nothing prevents the scheduler from placing both replicas on the same node in the first place. When that happens, kubelet just bind-mounts the same volume path into both containers. No attach conflict (it's one node, one attach), no admission rejection (RWO doesn't check pod identity), no error anywhere. Two processes now have concurrent read-write access to the same filesystem, and whether that corrupts anything depends entirely on whether the application was written to coordinate that itself — which most stateful software (databases, etcd, Kafka brokers) is not.

This is precisely why `ReadWriteOncePod` had to be added as a *new*, fourth mode rather than just tightening the existing behavior of RWO — changing RWO's own semantics after the fact would have been a breaking change for every existing workload relying on "any pod on this node can mount it." RWOP is opt-in specifically for workloads that need the harder guarantee.

The actual fix for this pitfall is architectural, not a flag: a truly single-instance stateful workload belongs on a `StatefulSet` (one PVC per replica, via `volumeClaimTemplates` — replicas can never collide on a volume because they never share one), and genuinely-concurrent access belongs on an RWX-capable backend (EFS), not RWO.

---

## Dynamic Provisioning Walkthrough: A GP3 PVC, End to End

This traces the exact sequence of controllers, informers, and API objects behind the common case: a PVC requesting `gp3` (AWS EBS) storage via a `WaitForFirstConsumer` StorageClass, referenced by a pod.

### The players

Kubernetes doesn't provision or attach storage itself — it defines the **Container Storage Interface (CSI)**, an interface analogous to the CRI that `kubelet.md` covers for container runtimes: core Kubernetes owns the contract, a vendor-supplied driver owns the implementation. For EBS, that driver is `ebs.csi.aws.com`.

| Actor | Where it runs | What it watches | What it does |
|---|---|---|---|
| **Scheduler** (`VolumeBinding` plugin) | `kube-scheduler`, control plane | Pods being scheduled, PVCs they reference | Picks a candidate node; if a referenced PVC is unbound and `WaitForFirstConsumer`, stamps the PVC with the node choice and stalls the pod's own bind until the PVC resolves |
| **external-provisioner** | CSI sidecar container, runs as a Deployment/StatefulSet pod installed alongside the EBS CSI driver | PVCs (via informer) | On seeing a PVC with a matching `storageClassName` and no `volumeName`, calls the driver's `CreateVolume` gRPC; on success, creates the PV object itself, with `spec.claimRef` pre-set to the requesting PVC |
| **EBS CSI driver** | Container in the same pod as external-provisioner, talked to over a local Unix socket | N/A (gRPC server, not a watcher) | Implements `CreateVolume` — calls the AWS EC2 API to actually create the EBS volume, returns the volume ID |
| **PV controller** | `kube-controller-manager`, control plane | PVs and PVCs (via informer) | Sees the new PV already carries a `claimRef`; sets `PVC.spec.volumeName` to match, flips both to `Bound`. No search/match logic runs here — that logic (line ~61 above) is for *static* provisioning only, where a PV really is unclaimed |

### Manifests in play

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  storageClassName: gp3
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 20Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers: [...]
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data
```

### The sequence

```
1. PVC "data" created, storageClassName: gp3, no volumeName
       │
       ▼
2. external-provisioner's informer fires on the new PVC.
   StorageClass says WaitForFirstConsumer + no selected-node
   annotation yet → provisioner takes no action, PVC stays Pending
   with condition "waiting for first consumer to be created before binding"
       │
       ▼
3. Pod "app" created, references PVC "data"
       │
       ▼
4. Scheduler begins the scheduling cycle for "app": Filter/Score
   nodes on ordinary constraints (resources, taints, affinity —
   NOT volume topology yet, there's no volume)
       │
       ▼
5. VolumeBinding plugin's Reserve phase tentatively picks node N.
   Writes annotation on the PVC:
     volume.kubernetes.io/selected-node: N
   Scheduler's PreBind phase now blocks, waiting on this PVC to
   reach Bound — the pod is NOT yet bound to N
       │
       ▼
6. external-provisioner's informer fires again on the PVC update.
   Sees the selected-node annotation → calls CreateVolume via gRPC
   to the EBS CSI driver, passing N's zone as a topology requirement
       │
       ▼
7. EBS CSI driver calls the AWS EC2 API → EBS volume created in N's
   zone → volume ID returned
       │
       ▼
8. external-provisioner creates the PV object:
     spec.csi.volumeHandle = <ebs-volume-id>
     spec.claimRef = {name: data, namespace: ..., uid: ...}   ← pre-set here
       │
       ▼
9. PV controller's informer fires on the new PV. claimRef already
   names a specific PVC — no matching search needed. Sets:
     PVC.spec.volumeName = <pv-name>
   Both objects flip to Bound
       │
       ▼
10. Scheduler's PreBind phase (blocked since step 5) sees the PVC
    is Bound → unblocks → proceeds with the actual pod-to-node bind
       │
       ▼
11. Pod "app" is bound to node N, kubelet starts the volume mount
```

The key structural insight: nothing here does an open-ended search for a match. Every step is a direct pointer to the next — the annotation names the node, `claimRef` names the PVC, `volumeName` names the PV. Matching-by-search is entirely a static-provisioning concern.

---

### Finalizers

Finalizers are markers on a resource that block its deletion until a controller confirms it is safe to remove.

| Resource | Finalizer | Purpose |
|---|---|---|
| PVC | `kubernetes.io/pvc-protection` | Blocks PVC deletion while a pod is actively mounting it |
| PV | `kubernetes.io/pv-protection` | Blocks PV deletion while it is bound to a PVC |

Both are managed automatically by the **PVC/PV Protection Controller** — you do not set them manually.

---

### How Finalizers Affect Pending States

Finalizers can extend "blocked" states and cause a cascade of `Pending` or `Terminating` resources:

**Scenario 1 — Deleting a PVC while a pod is using it:**

```
kubectl delete pvc my-pvc
    │
    ▼
deletionTimestamp set on PVC
    │
    ▼
PVC enters Terminating (finalizer holds it)
    │
    ▼
New pods referencing this PVC → stay Pending
    │
    ▼
Pod deleted → finalizer removed → PVC fully deleted
```

**Scenario 2 — PV still considered bound:**

```
PVC deleted but PV finalizer not yet cleared
    │
    ▼
PV stays Bound (or Released with claimRef)
    │
    ▼
New PVC cannot bind to it → stays Pending
    │
    ▼
Pod referencing new PVC → stays Pending
```

> If something is stuck in `Pending` or `Terminating`, check the full chain: is a finalizer blocking cleanup at a lower layer?

```bash
kubectl describe pvc <name>   # check Finalizers and Events
kubectl describe pv <name>    # check Finalizers and claimRef
```

---

## Static Provisioning Walkthrough: A Local PV, `WaitForFirstConsumer`

The GP3 walkthrough above is the dynamic-provisioning path. This is the other one: a PV that's **already fully formed** before any PVC exists, pinned to one specific node. Structurally different in one key way — there is no external-provisioner in this path at all, nothing ever calls `CreateVolume`.

### The setup

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-node7
spec:
  capacity:
    storage: 100Gi
  accessModes: [ReadWriteOnce]
  storageClassName: local-fast
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values: [node-7]
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-fast
provisioner: kubernetes.io/no-provisioner   # nothing dynamically creates local volumes
volumeBindingMode: WaitForFirstConsumer
```

`local-pv-node7` was created by an admin (or a helper like the community `local-static-provisioner`, which just automates writing PV manifests like this one for disks it discovers — it doesn't change anything about the binding mechanics below). Its `nodeAffinity` is the load-bearing field: the data physically only exists on `node-7`.

### What `WaitForFirstConsumer` is actually deferring here

With no provisioning step, it isn't "wait for a volume to be created." It's "don't let the PV controller's normal eager static bind-by-search run before the scheduler has weighed in." Under `Immediate`, the moment the PVC is created, the PV controller would run its ordinary static matching logic (`storageClassName` + `accessModes` + capacity — the same search described in "Binding Process" above), find `local-pv-node7` as the only match, and bind it — before anyone has checked whether a pod can actually land on `node-7`. `WaitForFirstConsumer` holds that bind back specifically so the scheduler can treat the PV's `nodeAffinity` as an input to scheduling, not a constraint discovered after the fact.

### The sequence

```
1. local-pv-node7 already exists, Available, nodeAffinity: node-7,
   storageClassName: local-fast
       │
       ▼
2. PVC "data" created, storageClassName: local-fast, no volumeName.
   WaitForFirstConsumer → PV controller takes no action.
   PVC stays Pending: "waiting for first consumer to be created
   before binding" — same condition as the dynamic case, same reason:
   nothing tries to bind an unconsumed WaitForFirstConsumer PVC
       │
       ▼
3. Pod "app" created, references PVC "data"
       │
       ▼
4. Scheduler's Filter phase runs, per candidate node, jointly with
   every other filter (resources, taints, affinity). For THIS pod's
   unbound PVC, the VolumeBinding plugin asks per node: "is there an
   Available PV of this StorageClass whose nodeAffinity permits this
   node?"
     - node-5, node-12, etc. → no matching PV → fail Filter
     - node-7 → local-pv-node7 matches → passes Filter
   (If more than one local PV existed on different nodes, more than
   one node would pass here, and Scoring would pick among them.)
       │
       ▼
5. Reserve: plugin assumes local-pv-node7 ↔ PVC "data" in its cache
       │
       ▼
6. PreBind: unlike the dynamic case, there's no external-provisioner
   to wait on — the scheduler's own PreBind step performs the real
   bind directly: sets PV.spec.claimRef and PVC.spec.volumeName
   itself. Both flip to Bound immediately, no separate controller
   round-trip needed
       │
       ▼
7. Pod "app" bound to node-7, kubelet mounts /mnt/disks/ssd1
```

### Why `Immediate` would be worse here than for zonal EBS

Suppose `node-7` itself fails Filter for an unrelated reason — it's tainted, or out of CPU. Under `WaitForFirstConsumer`, *every* node fails Filter (the others for lacking a matching PV, node-7 for its own reason), so Reserve/PreBind never run. The PVC never binds, `local-pv-node7` never gets a `claimRef`. The pod sits in an ordinary, self-healing `Pending` — the ordinary "didn't find available persistent volumes to bind" case. The instant node-7's problem clears, the next scheduling attempt just succeeds, with no operator involved.

Under `Immediate`, the PV controller would have bound `local-pv-node7` to the PVC the moment the PVC was created — before anyone knew node-7 had a problem. The pod is stuck Pending exactly the same either way, but now the PV is also **permanently locked** to a PVC that can never actually be scheduled. There's no second local PV to fall back to (unlike the zonal-EBS `Immediate` pitfall, where other nodes in the zone might still work) — recovering requires an operator to notice, delete the PVC, and manually clear `claimRef` per the "Reusing a PV" procedure above. `WaitForFirstConsumer` doesn't prevent the scheduling failure in this scenario — it changes the failure mode from *a silently wasted, manually-recoverable PV* to *an ordinary, self-healing Pending pod*.

---

## Mental Model

```
PV          = the physical disk
PVC         = a request to use that disk
claimRef    = the lock on the disk (PV side)
finalizer   = the safety latch (prevents premature deletion)
Pod         = the consumer — blocked until everything below is ready

Delete PVC while pod runs  → PVC stuck Terminating, pod keeps working
Pod deleted                → finalizer removed, PVC fully deleted
PV Released + claimRef set → locked, new PVC cannot bind
Patch claimRef away        → PV becomes Available again

Something stuck Pending?   → check the full chain: PV → PVC → Pod
```

---

## DevOps Takeaways

- **Prefer dynamic provisioning** — less operational overhead, exact sizing
- **Use `Retain` only when you need manual control** — local volumes, data recovery, stateful workloads where data must survive PVC deletion
- **Always inspect data before reusing** a released PV
- **Always use `spec.volumeName`** when rebinding to a specific PV — never rely on automatic selection
- **Static provisioning requires upfront capacity planning** — overprovisioned PVs waste storage

---

## What This Covers So Far

This covers the PV/PVC binding model and its controller (`kube-controller-manager`'s PV controller), static vs. dynamic provisioning, reclaim policies and the manual `Retain`-reuse procedure, the finalizer-driven pending/terminating cascades, `volumeBindingMode`'s two values and the `Immediate` zone-affinity-conflict pitfall, the four access modes and which distinct actor enforces each (attach/detach controller for RWO, kubelet for ROX, the CSI driver's own capability negotiation for RWX, kube-apiserver admission for RWOP) plus the `Deployment`-sharing-an-RWO-PVC pitfall in both its loud and silent forms, the full dynamic-provisioning sequence for a `WaitForFirstConsumer` GP3 PVC across all four actors (scheduler's `VolumeBinding` plugin, external-provisioner, CSI driver, PV controller), and the static-provisioning counterpart for a local PV — where `Filter` runs the node-affinity match per candidate node instead of a provisioner reacting to an annotation, and the scheduler's own `PreBind` step performs the bind directly since there's no external actor to wait on. It doesn't yet cover StorageClass fields beyond `provisioner` and `volumeBindingMode` (`allowVolumeExpansion`, `allowedTopologies`, `mountOptions`, and which of these are actually mutable post-creation vs. immutable), what happens on `CreateVolume` failure/retry (exponential backoff, event surfacing on the PVC), the exact CSI `ControllerGetCapabilities` negotiation behind the RWX rejection, or how the community `local-static-provisioner` automates discovering and registering local disks as PVs versus the fully manual creation shown above.
