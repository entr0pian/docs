# Kubernetes ReplicaSets and Deployments

Covers two related things: how a Deployment's update/rollback plays out across ReplicaSets (below), and — further down — how the ReplicaSet controller itself actually reconciles Pods, avoids overshooting on Pod creation, and why it deliberately doesn't enforce template conformance on Pods that already exist.

---

## Core Concept

A **Deployment** orchestrates **ReplicaSets**. ReplicaSets maintain **Pods**.

Every change to a Deployment's `PodTemplateSpec` produces a new ReplicaSet — the old one is kept at 0 replicas as a rollback revision. Kubernetes never mutates a ReplicaSet in-place; it always creates a new one.

> **Key Idea:** Every Deployment revision = a distinct ReplicaSet. Rollback = re-scaling an old ReplicaSet back up.

---

## Update Flow

### 1. Initial State

```
Deployment: myapp
ReplicaSet:  myapp-7d6f5d8b7   →   3 × nginx:1.0
```

### 2. User Triggers Update

```bash
# e.g. image change: nginx:1.0 → nginx:2.0
kubectl apply -f deployment.yaml
```

### 3. Deployment Controller Detects the Change

`PodTemplateSpec` has changed. Because ReplicaSets are treated as **immutable rollout revisions**, the controller does **not** modify the existing ReplicaSet — it creates a new one.

```
Old RS:  myapp-7d6f5d8b7   (nginx:1.0)
New RS:  myapp-6f4c9b77d   (nginx:2.0)  ← created, starts at 0 pods
```

### 4. Rolling Update

Example strategy: `maxSurge: 1`, `maxUnavailable: 1`, `desired: 3`

```
Start       Old RS = 3   New RS = 0
Step 1      Old RS = 2   New RS = 1   (total: 3)
            ↳ wait until new Pod is Ready
Step 2      Old RS = 1   New RS = 2
            ↳ wait until new Pod is Ready
Final       Old RS = 0   New RS = 3
```

The controller always waits for each new Pod to reach **Ready** before continuing.

### 5. After Successful Rollout

```
Active RS:  myapp-6f4c9b77d   →   3 × nginx:2.0
Old RS:     myapp-7d6f5d8b7   →   0 pods  (retained for rollback history)
```

---

## Rollback Flow

### 1. User Triggers Rollback

```bash
kubectl rollout undo deployment myapp
```

### 2. Deployment Finds the Previous Revision

```
Current:   myapp-6f4c9b77d  →  nginx:2.0
Previous:  myapp-7d6f5d8b7  →  nginx:1.0
```

### 3. Scaling is Reversed

```
Start       v2 = 3   v1 = 0
Step 1      v2 = 2   v1 = 1
Step 2      v2 = 1   v1 = 2
Final       v2 = 0   v1 = 3
```

The same rolling mechanism applies — Kubernetes does not modify running Pods or downgrade containers in-place. Old Pods are recreated from the previous ReplicaSet; current Pods are terminated.

> **Important:** Rollback is just another rolling update — it reuses the existing old ReplicaSet rather than creating a new one.

---

## Rollback History

By default, Kubernetes retains the last **10** ReplicaSet revisions (`revisionHistoryLimit`). To inspect:

```bash
kubectl rollout history deployment myapp
kubectl rollout history deployment myapp --revision=2
kubectl rollout undo deployment myapp --to-revision=2   # roll back to a specific revision
```

---

## Summary

| Concept | Behaviour |
|---|---|
| `PodTemplateSpec` change | Always creates a new ReplicaSet |
| Old ReplicaSet | Kept at 0 replicas — never deleted (up to `revisionHistoryLimit`) |
| Rollback | Re-scales an existing old RS; does not create a new one |
| Pod mutation | Never — Kubernetes replaces, it does not patch running Pods |
| Rollback mechanism | Identical rolling update logic, just in reverse direction |

---

## How the ReplicaSet Controller Actually Reconciles

Everything above is the Deployment controller's view: it swaps whole ReplicaSet objects and shifts replica counts between them. Underneath, each individual ReplicaSet is kept at its assigned count by the **ReplicaSet controller**, which is a much dumber, narrower loop than it looks from the outside.

It watches Pods through a **pod informer**, not just ReplicaSet objects. Every Pod create/update/delete event is handled by resolving which ReplicaSet (if any) owns that Pod, and enqueuing **that ReplicaSet's key** (`namespace/name`) onto a workqueue — never the Pod itself. The workqueue carries no per-Pod facts; a burst of several Pod-delete events for the same RS collapses into that one RS key being (re-)enqueued, since the queue de-dupes.

Ownership is resolved two ways:

- **`ownerReferences`** — a Pod created by this RS already carries a controller reference back to it.
- **Label-selector adoption/orphaning** — a Pod with no controller ref but matching labels gets adopted (an ownerRef is added). A Pod that stops matching the RS's selector (e.g. its labels are edited) gets released — ownerRef stripped — but is **not deleted** by the RS. It's simply no longer counted. This is also the mechanism behind the classic "isolate one Pod for live debugging" trick: `kubectl label pod foo app-` strips it from the Deployment/RS without killing it.

When a worker dequeues an RS key, `syncReplicaSet` does a **full recount from the local informer cache**: list every Pod matching the selector, filter to non-terminal ones (phase not `Succeeded`/`Failed`, no deletion timestamp), and compare that count to `.spec.replicas`. There's no per-event bookkeeping like "this specific Pod died, spawn its replacement" — every sync recomputes the diff from scratch.

Two consequences worth being explicit about:

- **Readiness is irrelevant to this count.** A `Running` Pod that's permanently failing its `readinessProbe` still counts as one of the N replicas — the controller has no reason to replace it. Traffic membership is a separate concern entirely, owned by the EndpointSlice controller (see `pod_lifecycle_and_restarts.md`).
- **The Pod template is consulted only when creating a replacement.** It plays no role in evaluating Pods that already exist and still match the selector — which is the subject of the next section.

---

## Why the ReplicaSet Doesn't Enforce Template Conformance on Existing Pods

A natural expectation: if a Pod's spec drifts from `.spec.template`, shouldn't the RS notice and fix it? It doesn't, and that's deliberate:

1. **Most of the Pod spec is immutable by API-server validation once a Pod is admitted** — env vars, volumes, resource requests, scheduling fields, and more all reject in-place writes outright. There's very little live drift to detect in the first place. `application_configuration.md`'s "Pod Spec Immutability" table has the exact mutable-field list: `metadata.labels`/`annotations`, `spec.containers[*].image`, `spec.tolerations` (additions only), `spec.activeDeadlineSeconds`, and CPU/memory via in-place resize.
2. **The fields that are mutable are mutable on purpose**, as operator escape hatches — e.g. patching one Pod's image for a live debug session without a full rollout. A controller that reverted any deviation from the template within milliseconds would make that capability pointless.
3. **Template changes are handled at a different, coarser granularity.** A given RS's own template never changes after creation. When the Deployment's template changes, the Deployment controller creates an entirely new RS object carrying the new template and shifts `.spec.replicas` between old and new (the Update Flow above). Conformance is enforced by *object substitution*, once, atomically — not by continuous per-Pod diffing.
4. **Cost.** Deep-diffing every owned Pod's live spec against the template on every reconcile, cluster-wide, is real API-server and CPU load for a benefit that's mostly moot given point 1. The ReplicaSet's reconcile loop is deliberately the cheapest invariant that does the job: `len(list) == int`.
5. **It's relied upon operationally**, not just tolerated — see the orphaning trick above. It only works because the RS's ownership/counting logic cares exclusively about selector match, nothing else.

The RS's actual guarantee is narrower than it first appears: **N non-terminal Pods matching the selector**, not **N Pods running this exact template**. The template only governs Pods the RS itself creates.

---

## Avoiding Overshoot: the Expectations Mechanism

Every sync recomputes its diff from a **local, potentially stale informer cache**, not a live read against the API server. Acting on that diff naively every time a Pod event fires risks overshoot — e.g. issuing a second batch of creates before the cache has caught up on the first.

The fix is an in-memory per-RS counter called an **expectation**:

1. `syncReplicaSet` recounts and decides it needs, say, 5 more Pods.
2. **Before** issuing any `Create` calls, it records `ExpectCreations(rsKey, 5)` — "I'm about to ask for 5 creates; don't trust a recount for this RS again until I've observed all 5 land."
3. It issues the 5 `Create` calls, batched via **slow start** (1, then 2, then 4... doubling, capped) so a large scale-up doesn't hit the API server as one giant burst.
4. At the top of every `syncReplicaSet` call, it checks whether the RS's expectations are satisfied. If not, the create/delete decision logic is skipped entirely for this sync — only a status update happens. Re-enqueues of the same RS key are a no-op on the create/delete side until the previous batch is confirmed.
5. The pod informer firing an add event for a new Pod owned by this RS decrements the counter by 1. For creates this is a **plain count**, not tied to a specific Pod ID — the Pod's name/UID doesn't exist until the `Create` call returns.
6. **Deletes are UID-tracked**, unlike creates: `ExpectDeletions(rsKey, [uid1..uid5])` records the exact UIDs targeted, since those Pods already exist and their UIDs are known before the call goes out. This disambiguates "the Pod I asked to be deleted is actually gone" from "some unrelated Pod under this RS happened to change right now."
7. If a `Create`/`Delete` call fails synchronously (e.g. a ResourceQuota rejection), the expectation is decremented immediately at the failure site — the controller doesn't wait for an informer event that will never arrive.
8. A several-minute timeout is the fallback safety net for the rare case where an expected event genuinely never shows up. It's not the normal mechanism — resolution is event-driven and typically near-instant.

---

## What Triggers Pod Replacement (and What Doesn't)

| Event | RS creates a replacement? | Why |
|---|---|---|
| Pod object deleted (manual, evicted, preempted, node-pressure) | Yes | Active count drops below `.spec.replicas` |
| Pod's labels no longer match the RS selector | Yes (orphaned original keeps running, unmanaged) | No longer counted as active for this RS |
| Container image (or other mutable field) edited directly on a live Pod | No | RS never re-evaluates spec conformance on existing Pods |
| `livenessProbe` failure | No — container restarts in place, same Pod/UID | Kubelet's job, not the RS's; `restartPolicy: Always` is mandatory for controller-owned Pods |
| `readinessProbe` failure | No | Pod is pulled from the Service's EndpointSlice; RS still sees it as `Running`/non-terminal |
| Node-pressure eviction | Yes | Kubelet deletes the whole Pod to protect the Node; same delete-detection path as any other deletion |

---

## What This Covers So Far

Open threads from this session, not yet traced:

- The exact slow-start batch sizes/cap, and whether they differ for creates vs. deletes.
- Which specific Pod the RS controller picks to delete first during a scale-down when several are equally "active but unready" — there's a documented ranking (unscheduled, then pending, then not-ready, then younger) this doc doesn't walk through yet.
- How the Deployment controller's own reconcile loop decides step-by-step surge/unavailable counts across old and new RS during a rolling update — this doc shows the end states, not the per-step decision logic.
- StatefulSet's very different update model (in-place ordinal updates, no parallel new-object substitution) as a contrast case to everything above.
- How `PodDisruptionBudget` gates the Eviction API path specifically, versus the other deletion triggers in the table above.
- In-place Pod vertical scaling's interaction with the expectations mechanism — does a resize count as a "create" or "delete" expectation, or neither?
