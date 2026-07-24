# cAdvisor Metrics, the Pause Container, and Pod-Level Aggregates

## Overview

When Prometheus scrapes container metrics in Kubernetes it talks to **cAdvisor**, which is embedded inside every `kubelet`. Understanding what cAdvisor exposes — and why — explains the filters used in Prometheus alerting rules:

```promql
container!=""
container!="POD"
```

---

## cAdvisor and its Role

cAdvisor (Container Advisor) is an open-source daemon originally built by Google. In Kubernetes it runs as part of the `kubelet` process on every node and is responsible for collecting resource usage and performance characteristics of every running container.

It exposes metrics at `/metrics/cadvisor` on the kubelet's HTTPS port (typically `10250`). The Prometheus operator scrapes this endpoint via the `kubelet` ServiceMonitor and stores the data under metric families such as:

- `container_cpu_usage_seconds_total`
- `container_memory_working_set_bytes`
- `container_fs_reads_bytes_total`
- `container_network_receive_bytes_total`

Every time series emitted by cAdvisor carries labels that identify what it is measuring: `namespace`, `pod`, `container`, `image`, and `node`.

---

## The Three Layers cAdvisor Measures

For every pod, cAdvisor emits metrics at three distinct scopes. This is the root cause of the label noise.

### 1. Individual Containers (the useful data)

Each container defined in a pod spec gets its own time series with `container="<name>"`.

```
container_cpu_usage_seconds_total{pod="taskapp-backend-abc", container="backend"}
```

This is the data you actually want for alerting.

### 2. Pod-Level Aggregate (`container=""`)

cAdvisor also emits one time series per pod where the `container` label is an **empty string**. This series represents the **sum of all containers in the pod** — it is a rolled-up aggregate produced by cAdvisor itself, not by Prometheus.

```
container_cpu_usage_seconds_total{pod="taskapp-backend-abc", container=""}
```

Why does cAdvisor do this? Because the Linux kernel tracks resource usage at the **cgroup** level. A pod maps to a parent cgroup, and each container maps to a child cgroup nested beneath it. cAdvisor walks the entire cgroup tree and emits a series for every node in that tree — including the pod-level parent cgroup.

**If you do not filter `container!=""` your alert expression would double-count** — you would be evaluating both per-container values and the aggregate, potentially triggering alerts on the same workload twice or computing incorrect averages.

### 3. The Pause Container (`container="POD"`)

Every Kubernetes pod contains a hidden container called the **pause container** (also referred to as the infra container or sandbox container). It is injected automatically by the kubelet — it never appears in your pod spec.

```
container_cpu_usage_seconds_total{pod="taskapp-backend-abc", container="POD"}
```

#### What does the pause container do?

The pause container runs a single binary (`/pause`) that immediately suspends itself after startup. Its only job is to **hold the Linux namespaces** that all other containers in the pod share:

- **Network namespace** — all containers share the same IP address and port space
- **IPC namespace** — enables inter-process communication between containers
- **PID namespace** (optional) — shared process tree

Because the pause container owns the namespaces, they persist even if an application container crashes and is restarted. Without the pause container every restart would destroy and recreate the network namespace, which would change the pod's IP address.

#### Why does cAdvisor give it the label `"POD"`?

cAdvisor identifies the pause container by its image (`registry.k8s.io/pause`) and assigns the special label value `"POD"` rather than a real container name. This distinguishes it from application containers.

Its resource consumption is negligible (a few KiB of memory, near-zero CPU), so it is meaningless for workload alerting. Including it would add noise and slightly skew utilization calculations upward.

---

## How the Filters Work Together

```promql
container_memory_working_set_bytes{
  namespace="default",
  container!="",     # drops the pod-level aggregate cgroup series
  container!="POD"   # drops the pause/infra container series
}
```

After applying both filters, the remaining time series represent exactly the application containers defined in the pod spec — the only data that is meaningful for alerting on workload health.

---

## Summary Table

| `container` label value | What it represents | Useful for alerting? |
|---|---|---|
| `"backend"`, `"nginx"`, etc. | Application container | Yes |
| `""` (empty string) | Pod-level cgroup aggregate | No — double-counts |
| `"POD"` | Pause / infra container | No — negligible noise |
