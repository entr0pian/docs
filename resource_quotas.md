# Resource Quotas

## What a ResourceQuota actually is

A ResourceQuota is a namespace-level guardrail that limits how much of the cluster a namespace can consume.

Think of it as:

> "This namespace is allowed to use up to X resources — no more."

- It doesn't assign resources.
- It just blocks new resource creation when limits are exceeded.

---

## What can you limit?

There are 3 main categories:

1. **Compute resources** (most common)
   - CPU (`requests.cpu`, `limits.cpu`)
   - Memory (`requests.memory`, `limits.memory`)
2. **Object counts**
   - Pods
   - Services
   - ConfigMaps
   - Secrets
   - PVCs
3. **Storage**
   - PersistentVolumeClaims storage
   - StorageClass-specific quotas

---

## Example: Basic ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "10"
```

---

## How Kubernetes enforces it

When you create a Pod, Kubernetes calculates:

1. Sum of all existing resource usage in the namespace
2. The new Pod's requested resources

If it exceeds the quota → the request is rejected.

You'll see an error like:

```
exceeded quota: dev-quota, requested: requests.cpu=1, used: 4, limited: 4
```

---

## Important gotcha (very common)

> ResourceQuota works on **requests/limits**, NOT actual usage.

So this matters:

```yaml
resources:
  requests:
    cpu: 500m
```

---

## ResourceQuota + LimitRange (critical combo)

In real setups, you almost always combine:

1. **ResourceQuota** → max allowed per namespace
2. **LimitRange** → defaults + per-pod constraints

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
        cpu: 200m
        memory: 256Mi
      default:
        cpu: 500m
        memory: 512Mi
```

### Example output

```
kubectl describe quota compute-resources --namespace=myspace

Name:                    compute-resources
Namespace:               myspace
Resource                 Used  Hard
--------                 ----  ----
limits.cpu               0     2
limits.memory            0     2Gi
requests.cpu             0     1
requests.memory          0     1Gi
requests.nvidia.com/gpu  0     4
```

### Why this matters

- Ensures every Pod has requests/limits
- Prevents users from bypassing quotas

---

## Common use cases

**Multi-team clusters**
- Each team gets a namespace
- Quotas prevent one team from starving others

**Cost control (very relevant in EKS)**
- Indirectly limits how much compute can be scheduled
- Helps avoid "oops we scaled 200 pods"

**Platform vs application separation**
- `platform` namespace → large quota
- `dev-*` namespaces → strict quotas

**CI/CD environments**
- Limit number of ephemeral environments (pods, PVCs)

---

## What ResourceQuota does NOT do

- Does **not** guarantee resources (the scheduler does that)
- Does **not** prevent runtime overuse (use limits for that)
- Does **not** control who can create resources (RBAC does that)

---

## Quota for specific resource types

`kubectl` supports object count quota for all standard namespaced resources using the syntax `count/<resource>.<group>`:

```bash
kubectl create quota test \
  --hard=count/deployments.apps=2,count/replicasets.apps=4,count/pods=3,count/secrets=4 \
  --namespace=myspace
```

```yaml
count/deployments.apps: "5"
count/services: "10"
```

### Example output

```
Name:                         test
Namespace:                    myspace
Resource                      Used  Hard
--------                      ----  ----
count/deployments.apps        1     2
count/pods                    2     3
count/replicasets.apps        1     4
count/secrets                 1     4
```
