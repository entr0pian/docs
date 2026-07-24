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

The scheduler is responsible for placing a pod on a node. Storage availability is part of that decision.

- For `volumeBindingMode: Immediate` — PVC binds as soon as it is created, independently of scheduling. The pod is placed on any node.
- For `volumeBindingMode: WaitForFirstConsumer` — PVC binding is **deferred until a pod is scheduled**. The scheduler picks a node first, then the PV is provisioned or selected based on that node's topology (important for local volumes and zone-aware storage).

| Binding Mode | PV Provisioned When | Use Case |
|---|---|---|
| `Immediate` | PVC is created | Cloud block storage, NFS |
| `WaitForFirstConsumer` | Pod is scheduled to a node | Local volumes, zone-pinned storage |

> Scheduling and volume binding are interconnected — a wrong binding mode for local storage can cause pods to be scheduled on nodes where the volume doesn't exist.

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
