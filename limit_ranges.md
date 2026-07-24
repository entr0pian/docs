# Kubernetes LimitRange

## Overview

A `LimitRange` is a namespace-scoped admission policy that serves two distinct roles:

1. **Mutation** — injects default `requests` and `limits` into containers that don't specify them
2. **Validation** — enforces minimum, maximum, and ratio constraints, rejecting non-compliant resources

### Admission Flow

```
User YAML → LimitRange Mutation → LimitRange Validation → ResourceQuota Check → API accepted/rejected
```

---

## Structure

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: example-limits
  namespace: dev
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 200m
        memory: 256Mi
      default:
        cpu: 500m
        memory: 512Mi
      min:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "2"
        memory: 2Gi
      maxLimitRequestRatio:
        cpu: "4"
```

---

## Field Reference

| Field | Description |
|---|---|
| `type` | Scope of the policy: `Container`, `Pod`, or `PersistentVolumeClaim` |
| `defaultRequest` | Injected as `requests` when none are specified. Used by the scheduler and autoscalers. |
| `default` | Injected as `limits` when none are specified. Enforced at runtime via cgroups. |
| `min` | Minimum allowed value for requests/limits. Prevents undersized or invalid configs. |
| `max` | Maximum allowed value for requests/limits. Prevents excessive resource usage. |
| `maxLimitRequestRatio` | Enforces `limit / request <= ratio`. Prevents extreme CPU bursting. |

### `type` Values

| Type | Scope |
|---|---|
| `Container` | Applied per container (most common) |
| `Pod` | Applied to the sum of all containers in the pod |
| `PersistentVolumeClaim` | Applied to storage requests |

---

## Behavior Examples

### 1. Missing Requests and Limits

When a container defines no resource block, the LimitRange injects defaults:

```yaml
# Input — no resources defined
containers:
  - name: app
    image: nginx
```

```yaml
# Result after LimitRange mutation
containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
```

### 2. Request Exceeds `max`

```yaml
resources:
  requests:
    cpu: "4"   # max is 2 → rejected
```

**Result:** `403 Forbidden` — `maximum cpu usage per Container is 2, but limit is 4`

### 3. Request Below `min`

```yaml
resources:
  requests:
    cpu: 50m   # min is 100m → rejected
```

**Result:** rejected — request is below the minimum allowed value.

### 4. `maxLimitRequestRatio` Violation

```yaml
resources:
  requests:
    cpu: 100m
  limits:
    cpu: 2000m  # ratio = 20x, allowed max is 4x → rejected
```

**Result:** rejected — `cpu max limit to request ratio per Container is 4, but provided ratio is 20`.

---

## Interactions

### LimitRange + ResourceQuota

LimitRange admission runs **before** ResourceQuota enforcement. This ordering is critical:

- Without LimitRange, pods without `requests` are not counted against the quota
- With LimitRange, defaults are injected first, ensuring every pod has requests/limits
- This makes ResourceQuota reliable and predictable

| Scenario | Quota Behavior |
|---|---|
| No LimitRange, pod has no requests | Quota not consumed — pod is `BestEffort` QoS |
| LimitRange present | Defaults injected → quota correctly consumed |

### LimitRange + Scheduler

The scheduler places pods exclusively based on `requests`, not `limits`.

- **`defaultRequest` too high** → pods may fail to schedule (insufficient allocatable resources)
- **`defaultRequest` too low** → nodes become overcommitted, risking OOM evictions

### LimitRange + HPA / KEDA

Autoscalers compute CPU utilization as:

```
utilization = actual usage / request
```

| Request Value | Effect on Autoscaling |
|---|---|
| Too high | Utilization stays low → scale-out never triggers |
| Too low | Utilization is always high → aggressive, unstable scaling |

Tuning `defaultRequest` is directly tuning your autoscaling sensitivity.

---

## Environment Design Reference

### Dev Namespace

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      default:
        cpu: 300m
        memory: 256Mi
      max:
        cpu: "1"
        memory: 1Gi
```

**Goal:** flexibility for development workloads with cost guardrails.

### Prod Namespace

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: prod-limits
  namespace: prod
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 300m
        memory: 256Mi
      default:
        cpu: 500m
        memory: 512Mi
      max:
        cpu: "2"
        memory: 4Gi
      maxLimitRequestRatio:
        cpu: "2"
```

**Goal:** stability and predictability. Strict ratio prevents burst overload; higher defaults reduce scheduling surprises.

---

## Common Pitfalls

| Symptom | Root Cause |
|---|---|
| Pod uses more resources than expected | LimitRange silently injected higher defaults |
| Pods stuck in `Pending` | `defaultRequest` is too high for available node capacity |
| HPA not scaling out | Request too high → utilization appears low |
| HPA scaling too aggressively | Request too low → utilization always appears high |
| ResourceQuota exceeded after adding LimitRange | Injected defaults now count against namespace quota |

---

## Mental Model

```
LimitRange   = per-container shape policy (mutation + validation)
ResourceQuota = namespace-wide budget (enforced after LimitRange)

Requests     = scheduling signal + autoscaling baseline
Limits       = runtime enforcement via cgroups
```

> **Key Takeaway:** LimitRange defines the _shape_ and _behavior_ of workloads in a namespace — not just their upper bounds. Getting requests right is as important as getting limits right.
