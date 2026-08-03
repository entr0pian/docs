# Prometheus Monitoring: Scrape Architecture, kube-state-metrics, and the Operator

Covers how CPU/memory and cluster-state metrics actually get from a node into
Grafana via Prometheus — the scrape targets involved, how Prometheus finds
them, and how the Prometheus Operator wires a target like kube-state-metrics
in end to end. Alerting (`PrometheusRule` + Alertmanager), PromQL query
construction, and Prometheus's own storage/scaling are separate enough to
earn their own sections here later rather than being folded in.

---

## Two Separate Systems, Not One

The first thing to get straight: Prometheus does **not** talk to the
Kubernetes Metrics API (`metrics.k8s.io`). That API is a different system
entirely — served by metrics-server via the API aggregation layer, backing
`kubectl top` and CPU/memory `HorizontalPodAutoscaler`s (see
`autoscaling.md`'s "The Metrics API: Aggregation, Not a Built-In Type"). It
holds no history, isn't in Prometheus format, and Prometheus has no reason to
scrape it.

Prometheus runs an entirely independent scrape path: it discovers targets
itself via the Kubernetes API and pulls `/metrics` endpoints in Prometheus's
own text exposition format on a fixed interval, storing every sample in its
own TSDB. The two systems happen to read overlapping data from the same
node — the kubelet — but through different endpoints, for different
consumers:

| | metrics-server | Prometheus |
|---|---|---|
| Kubelet endpoint | `/metrics/resource` (lightweight, CPU/mem only) | `/metrics/cadvisor` (full cAdvisor dump — cpu, mem, network, fs, per-cgroup) |
| Storage | None — in-memory, current value only | TSDB, full history |
| Served as | `metrics.k8s.io` (Kubernetes-typed API) | Native PromQL query API |
| Consumer | HPA, `kubectl top` | Grafana, alerting rules, Prometheus Adapter |

The one place these two paths *do* connect is Prometheus Adapter, and the
direction surprises people: it queries **Prometheus** via PromQL and
republishes the result under `custom.metrics.k8s.io` — Prometheus is
upstream of that API, never downstream of it. Full mechanics in
`autoscaling.md`'s "Prometheus Adapter: Its Exact Role".

---

## The Three Scrape Targets Behind "Cluster Monitoring"

A standard kube-prometheus-stack setup scrapes three structurally different
sources to answer "how healthy is my cluster":

1. **kubelet's cAdvisor** (`/metrics/cadvisor`) — per-container resource
   *usage*: `container_cpu_usage_seconds_total`,
   `container_memory_working_set_bytes`, etc. Built into the kubelet, no
   separate component to deploy. Label noise from the pod-level cgroup
   aggregate and the pause container is covered in `cadvisor_metrics.md`.
2. **node-exporter** — host-level metrics, not container-scoped at all:
   node CPU/mem/disk/network straight from `/proc` and `/sys`. This is not
   Kubernetes-aware; it would report the same thing running on a bare VM.
3. **kube-state-metrics** — Kubernetes object *state*, not usage:
   `kube_pod_status_phase`, `kube_deployment_status_replicas`,
   `kube_node_status_condition`, etc. Zero resource-usage numbers pass
   through it — that's cAdvisor's and node-exporter's job, not this one's.

---

## Where node-exporter and kube-state-metrics Come From

Neither ships with a vanilla cluster. Both are separate components you (or a
chart) deploy:

- **node-exporter** — a **DaemonSet**, one pod per node, running with
  `hostPath`/`hostPID` access to read the host's `/proc`/`/sys`.
- **kube-state-metrics** — a **Deployment** (single replica by default;
  shardable for very large clusters), watching the API server centrally
  rather than running per-node.

In practice you don't hand-roll either — the **kube-prometheus-stack** Helm
chart deploys both, along with Prometheus, the Operator, Alertmanager,
Grafana, and default `ServiceMonitor`s for the control plane (kubelet,
apiserver, scheduler, controller-manager, coredns). More on the bare
Operator vs. this chart below.

---

## kube-state-metrics: How It Actually Produces a Metric

Worth tracing because the mechanism isn't "query the API server on every
scrape" — that would make every Prometheus scrape interval hit the API
server directly, which doesn't scale.

Instead, on startup kube-state-metrics builds a `SharedInformerFactory`
(the same client-go machinery any controller uses) for each resource type
it's configured to track. Each informer does an initial `list`, then holds
a `watch` open, keeping a local in-memory cache incrementally in sync with
ADD/UPDATE/DELETE events. **When Prometheus scrapes it, no API call
happens at all** — the request just iterates the in-memory caches, and a
set of per-field "generator functions" turns cached object fields into
metric families on the fly (e.g. a Pod's `.status.phase` becomes
`kube_pod_status_phase{phase="Running"} 1`; every label on the object gets
copied into `kube_pod_labels`).

Two ports by default: `8080` serves the state metrics themselves, `8081`
(telemetry) serves kube-state-metrics' own process-health metrics. At
cluster scale, `--shard`/`--total-shards` splits the watched object set
across multiple instances by consistent-hashing on object UID, so one
instance's cache doesn't become a memory/latency bottleneck.

---

## How Prometheus Finds Its Targets

Prometheus doesn't get a static target list — it runs its own continuous
Kubernetes service discovery (`kubernetes_sd_configs`), watching the API
server (needs a `ClusterRole` with `list`/`watch` on `pods`, `nodes`,
`endpoints`, `services`), so new pods and nodes appear as scrape candidates
without a Prometheus restart. The configured `role` (`node`, `pod`,
`endpoints`, `endpointslice`, `service`) determines which object type is
watched and what `__meta_kubernetes_*` labels get attached to each
discovered target — those labels are what `relabel_configs` filters and
rewrites into the final scrape address and path.

Raw `scrape_configs`/`relabel_configs` is how you'd wire this by hand. The
Operator exists to generate that YAML for you from a higher-level object —
which is the actual point of this doc.

---

## The Prometheus Operator: Two Independent Label Matches

The Operator introduces CRDs — `ServiceMonitor`, `PodMonitor`, `Probe` — so
you declare *intent* ("scrape this Service's named port") instead of
hand-writing relabel rules. Getting a target to actually appear in
Prometheus depends on two **separate** label-matching relationships that
are easy to conflate:

1. **Prometheus CRD → ServiceMonitor.** The `Prometheus` object's
   `spec.serviceMonitorSelector` matches labels on the **ServiceMonitor's
   own `metadata.labels`**. If a `ServiceMonitor` doesn't carry a label this
   selector matches, the Operator ignores it completely — no error, it just
   never gets picked up.
2. **ServiceMonitor → Service.** The `ServiceMonitor`'s `spec.selector`
   matches labels on the **target Service's `metadata.labels`** — a
   different label set entirely, usually something like the app's own
   `app.kubernetes.io/name`.

A third gotcha in the same family: `spec.namespaceSelector` on the
`ServiceMonitor` — by default it only looks for matching Services in its
*own* namespace; a `ServiceMonitor` and its target Service in different
namespaces need `namespaceSelector.any: true` or an explicit namespace list.

### Worked Example: How kube-state-metrics Gets Wired Up, End to End

Representative of what the kube-prometheus-stack chart actually installs
(names/labels simplified for clarity):

**1. The Service**, with a named port:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-prometheus-stack-kube-state-metrics
  labels:
    app.kubernetes.io/name: kube-state-metrics
spec:
  ports:
    - name: http
      port: 8080
  selector:
    app.kubernetes.io/name: kube-state-metrics
```

**2. The ServiceMonitor**, satisfying both label relationships at once —
`release: kube-prometheus-stack` for the Operator to notice it,
`app.kubernetes.io/name: kube-state-metrics` to find the Service above:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kube-prometheus-stack-kube-state-metrics
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-state-metrics
  endpoints:
    - port: http        # the Service's named port, not a number
      interval: 30s
```

**3. The Prometheus CRD**, whose selector the ServiceMonitor's own labels
had to satisfy:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: kube-prometheus-stack-prometheus
spec:
  serviceMonitorSelector:
    matchLabels:
      release: kube-prometheus-stack
```

**4. What the Operator's control loop does with all three:** it watches
`Prometheus`, `ServiceMonitor`, `PodMonitor`, and `Probe` objects
cluster-wide. On any change, for each `Prometheus` object it lists every
`ServiceMonitor` matching that object's `serviceMonitorSelector` +
`namespaceSelector`; for each match, it resolves the `ServiceMonitor`'s own
`spec.selector` against Services (and their `Endpoints`) to find actual
IP:port targets; it converts each resolved `ServiceMonitor` + endpoint spec
into the equivalent native `scrape_configs`/`relabel_configs` stanza; and it
concatenates every resolved stanza from every matched `ServiceMonitor`/
`PodMonitor` into one complete `prometheus.yml` body.

**5. That generated config is written into a `Secret`** (something like
`prometheus-kube-prometheus-stack-prometheus`), mounted into the Prometheus
pod's filesystem.

**6. A sidecar container in that same pod — `prometheus-config-reloader`**
— watches the mounted file for changes and, on change, sends an HTTP `POST`
to Prometheus's own `/-/reload` endpoint. This only works because the
Operator starts Prometheus with `--web.enable-lifecycle`; without that flag
the endpoint is disabled. Prometheus reloads its config in-process — no pod
restart, no dropped scrape history.

**7. Within a few seconds**, `Status → Targets` in the Prometheus UI shows a
new job — named after the `ServiceMonitor`, something like
`serviceMonitor/<namespace>/kube-prometheus-stack-kube-state-metrics/0` —
with one target per matched `Endpoints` address:port.

No step in this chain is "Prometheus asks the API server for
kube-state-metrics" at scrape time — by the time a scrape happens, the
target list is already static YAML sitting on disk, refreshed only when the
CRDs actually change.

---

## Debugging a Missing Target

Given step 1/2 above fail *silently* on a label mismatch, don't just
re-read the CRD chain by hand — check Prometheus's own
**Status → Service Discovery** page first. It shows every target Prometheus
attempted to discover, including ones dropped by relabeling, which
immediately tells you whether the `ServiceMonitor` was picked up at all
(a CRD-selector problem) versus something further downstream (RBAC,
network policy, the target pod not actually listening).

---

## Bare Operator vs. kube-prometheus-stack

Worth being precise about, since it's easy to conflate: the Prometheus
**Operator** by itself is only the controller + CRDs (`Prometheus`,
`ServiceMonitor`, `PodMonitor`, `Probe`, `Alertmanager`) — it ships nothing
else. Installing just the Operator gets you an empty cluster with no
Prometheus instance, no kube-state-metrics, no `ServiceMonitor`s, nothing
to see in a UI.

**kube-prometheus-stack** is the Helm chart that bundles the Operator
*plus* a fully wired stack on top of it: Alertmanager, Grafana,
node-exporter (with its own `ServiceMonitor`), kube-state-metrics (with the
Service/ServiceMonitor pair above), a `Prometheus` CR, and default
`ServiceMonitor`s for the control plane — all pre-labeled consistently
(typically `release: <helm-release-name>`) so the two selector
relationships above are satisfied out of the box. That's the thing people
usually mean when they say "I installed Prometheus Operator."

---

## Grafana: Pull, Not Push

Grafana holds no data of its own for these dashboards — it's configured
with a data source pointing at Prometheus's Service (e.g.
`http://kube-prometheus-stack-prometheus:9090`). Each panel holds a PromQL
expression; on load, refresh, or time-range change, Grafana's backend
issues an HTTP request to Prometheus's query API (`/api/v1/query_range` for
time series, `/api/v1/query` for instant values) and renders whatever comes
back. Nothing is stored in Grafana between queries.

This keeps the whole chain pull-based end to end: Prometheus pulls from
kubelet/node-exporter/kube-state-metrics, Grafana pulls from Prometheus. The
one deliberate exception in the ecosystem is the **Pushgateway** — for
batch/cron jobs that finish and die before a scrape could ever catch them,
which push their result once and let Prometheus scrape the Pushgateway
itself like any other target. Not part of the chain above, but it's the
thing that breaks the "everything pulls" pattern.

---

## What This Covers So Far

This covers the scrape architecture end to end — what gets scraped, why
metrics-server and Prometheus read different kubelet endpoints, where
node-exporter and kube-state-metrics come from and how the latter actually
generates a metric, how Prometheus discovers targets, and the full
Operator CRD-watch → config-generate → hot-reload loop traced through a
concrete kube-state-metrics example. It doesn't yet cover: how a
`PodMonitor` differs in practice for apps with no fronting Service; the
`prometheus.io/scrape` annotation convention used by non-Operator setups
(the older, hand-rolled equivalent of a `ServiceMonitor`); alerting via
`PrometheusRule` + Alertmanager; actual PromQL construction for a
CPU/mem dashboard panel; or Prometheus's own storage model (TSDB, WAL,
retention) and how it scales past a single instance (federation, Thanos,
Mimir).
