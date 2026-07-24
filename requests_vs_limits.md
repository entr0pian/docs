# Kubernetes: Requests vs Limits

## Definitions

**Requests** — the minimum amount of CPU or memory guaranteed to a container. Used by the scheduler to decide which node a Pod can run on. The node must have at least this much available capacity.

**Limits** — the maximum amount of CPU or memory a container is allowed to consume. Enforced at runtime by the kernel via cgroups.

---

## Runtime Behavior

| Resource | Exceeds limit |
|---|---|
| CPU | Container is **throttled** (slowed down, not killed) |
| Memory | Container is **OOMKilled** (terminated immediately) |

The scheduler only considers **requests**, never limits. A node can be overcommitted — the sum of limits across Pods can exceed node capacity, but the sum of requests cannot.

---

## What Happens Without Them

| Configuration | Behavior |
|---|---|
| Neither set | Pod is `BestEffort` — no scheduling guarantee, evicted first under pressure |
| Only limits set | Kubernetes automatically sets `requests = limits` |
| Only requests set | No runtime cap — the container can burst to node capacity |

---

## QoS Classes

Kubernetes assigns a Quality of Service class based on how requests and limits are configured.

| Class | Condition | Priority |
|---|---|---|
| **Guaranteed** | `requests == limits` for all containers | Highest — last to be evicted |
| **Burstable** | `requests < limits` for at least one container | Middle — evicted if node is under pressure |
| **BestEffort** | No requests or limits set | Lowest — evicted first |

`Guaranteed` Pods are the most stable but also the least resource-efficient — they cannot burst beyond their allocation even if the node is idle.

---

## Autoscaling Implications (HPA / KEDA)

HPA computes CPU utilization as a percentage of **requests**, not limits:

```
utilization % = current usage / requests.cpu
```

- **Requests too low** → utilization % is inflated → HPA scales aggressively (false positives)
- **Requests too high** → utilization % is suppressed → HPA under-scales even under real load

Set requests close to the typical steady-state usage. Let limits absorb bursts.

---

## LimitRange and ResourceQuota Interaction

**LimitRange** — enforces per-Pod or per-container defaults and constraints within a namespace. If a container is submitted without requests/limits, LimitRange injects the defaults automatically.

**ResourceQuota** — caps the total aggregate consumption (sum of all requests or limits) across an entire namespace. Quotas compare against requests, not limits.

If a namespace has a ResourceQuota, every container **must** define requests and limits — otherwise the scheduler rejects the Pod.

---

## Best Practices

- Always define both requests and limits. Unlabelled Pods in `BestEffort` class are the first casualties during node pressure.
- Set `requests` based on observed typical usage (p50–p75 of actual consumption).
- Set `limits` based on the maximum you're willing to tolerate before killing the container (usually 2–4× requests, subject to LimitRange `maxLimitRequestRatio`).
- Pair with a LimitRange to prevent rogue Pods with no resource bounds landing in the namespace.
- If running HPA or KEDA, calibrate requests carefully — they are the denominator of your scaling signal.
