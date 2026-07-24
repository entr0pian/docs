# Kubernetes Deployment Update & Rollback Flow

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
