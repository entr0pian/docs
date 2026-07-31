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

This covers the PV/PVC binding model and its controller (`kube-controller-manager`'s PV controller), static vs. dynamic provisioning, reclaim policies and the manual `Retain`-reuse procedure, the finalizer-driven pending/terminating cascades, and — in depth — `volumeBindingMode`'s two values, the `Immediate` zone-affinity-conflict pitfall, and the full dynamic-provisioning sequence for a `WaitForFirstConsumer` GP3 PVC across all four actors (scheduler's `VolumeBinding` plugin, external-provisioner, CSI driver, PV controller). It doesn't yet cover the case flagged during that walkthrough: `WaitForFirstConsumer` applied to a **statically-provisioned local PV**, where there's no external-provisioner step at all — the disk already exists, pinned to one specific node via `nodeAffinity`, set by whoever created the PV up front. What does the `VolumeBinding` plugin actually do differently there — where does "waiting" come from with nothing to provision, and how does node selection work when the pod's other constraints have to somehow line up with wherever that one physical disk already lives? That's the seed for the next session on this doc. It also doesn't cover StorageClass fields beyond `provisioner` and `volumeBindingMode` (`allowVolumeExpansion`, `allowedTopologies`, `mountOptions`), or what happens on `CreateVolume` failure/retry (exponential backoff, event surfacing on the PVC).
