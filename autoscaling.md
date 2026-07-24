# Autoscaling: HPA and the Metrics API

Covers how an application scales to handle traffic fluctuations. This entry covers the HorizontalPodAutoscaler, the resource-metrics path that backs it, and the custom-metrics path via Prometheus Adapter — VerticalPodAutoscaler, KEDA, and Cluster Autoscaler are distinct enough topics to earn their own sections here later rather than a new file each.

---

## Is the HorizontalPodAutoscaler a Built-In Controller?

> **Question:** Does HPA ship with Kubernetes, or is it an add-on like a metrics stack?

Yes and no, and the split matters. The **HPA control loop** is one of the many reconcilers bundled inside the single `kube-controller-manager` binary — the same process running the Deployment controller, ReplicaSet controller, Node controller, and dozens of others. There is no separate "HPA operator" to install; if the control plane is up, that loop is already running.

What is **not** guaranteed to exist is anything for it to read. HPA depends entirely on a working `metrics.k8s.io` implementation being present in the cluster. Without one, `HorizontalPodAutoscaler` objects reconcile every cycle and simply report `<unknown>` for every metric — the controller isn't broken, it has nothing to query. Some managed distributions bundle a metrics backend by default, which is why it's easy to assume it's core; on a self-managed cluster it's an explicit install.

---

## How the HPA Controller Actually Reconciles

On a fixed sync period (`--horizontal-pod-autoscaler-sync-period`, 15s by default and not adjustable per-object), the controller re-evaluates every `HorizontalPodAutoscaler`. For each entry under `spec.metrics`, the metric's `type` determines which API group it queries:

| Metric type | API group queried |
|---|---|
| `Resource` (cpu/memory) | `metrics.k8s.io/v1beta1` |
| `Pods` / `Object` | `custom.metrics.k8s.io/v1beta2` |
| `External` | `external.metrics.k8s.io/v1beta1` |

Crucially, the controller has no config pointing at "the metrics server" anywhere — it issues a plain REST call like `GET /apis/metrics.k8s.io/v1beta1/namespaces/<ns>/pods?labelSelector=...` against the **normal kube-apiserver endpoint**, the same one any other client uses. It has no idea what's actually answering that group.

The scaling decision itself:

```
desiredReplicas = ceil(currentReplicas × (currentMetricValue / desiredMetricValue))
```

Scale-up and scale-down are deliberately asymmetric. `behavior.scaleDown.stabilizationWindowSeconds` (default 5 minutes) makes the controller look at the *maximum* recommended replica count over that whole window before actually scaling down, specifically to absorb noisy metrics without flapping. Scale-up has little to no such window by default — under-provisioning during a spike is treated as the worse failure mode than briefly over-provisioning.

**A metric choice worth being skeptical of:** memory is offered as a first-class `Resource` metric right alongside CPU, but it behaves very differently under scale-out. CPU usage is elastic — spreading load across more replicas genuinely reduces per-pod CPU pressure. Memory usually isn't; a pod's footprint (caches, connection pools, retained state) doesn't necessarily shrink just because siblings were added. Worse, a memory *leak* produces the same upward graph as legitimate load, so memory-based autoscaling can actively mask a leak by throwing more replicas at it instead of the pod ever hitting an OOM kill that would surface the bug.

---

## The Metrics API: Aggregation, Not a Built-In Type

`metrics.k8s.io` is not a type kube-apiserver knows how to serve on its own the way Pods or Deployments are — it's registered through the **API aggregation layer**, and understanding that mechanism is the key to understanding why the HPA controller can query it without knowing who's behind it.

An `APIService` object tells kube-apiserver: *"don't serve this group/version yourself — forward the entire request to this other Service instead."* A component inside kube-apiserver called `kube-aggregator` does the actual routing: it inspects the incoming request's group/version, matches it against a registered `APIService`, and proxies the whole HTTP call to whatever Service that object points at — trusting that backend's serving certificate via a `caBundle` field on the registration, the same shape of trust relationship a kubelet's serving cert requires of anything that calls it directly.

### Aggregated APIs vs CRDs: Two Different Extension Mechanisms

Both a `CustomResourceDefinition` and an `APIService` end up making a new path show up under `/apis/...`, which makes it tempting to treat them as the same kind of extension. They're not, and the difference is visible directly in the object shape:

- A **CRD** only ever describes a *schema*. There is no reference to any external server anywhere in it, because there doesn't need to be one — kube-apiserver's own generic apiserver machinery starts serving full CRUD for that type itself, backed by **etcd**, exactly like any built-in object. If you define a custom resource such as `Database`, apiserver is the one and only implementation that will ever exist for it; a controller elsewhere might watch and reconcile those objects, but it never serves the API.
- An **APIService** has a `service:` field pointing at a Service+namespace+port. There is a genuinely separate process behind it — an "extension apiserver" — that has to implement Kubernetes' List/Get/Watch semantics itself, using the same `k8s.io/apiserver` library a CRD gets for free, but running it manually in its own binary with its own serving cert. Nothing about the request ever touches etcd on this path.

**Why metrics-server needs the harder mechanism at all:** a CRD's whole model assumes the object should be *persisted* — written to etcd, versioned, watchable for changes. Resource-metric data is the opposite on every axis: it changes for every pod on every node roughly every 15 seconds, and nobody wants history retained, only the current value. Writing that volume of churn into etcd as CRD objects would abuse a store that isn't sized for that write rate, for data nobody needs kept anyway. Aggregation sidesteps this entirely — the extension apiserver can hold the numbers in memory and answer each request computed live, with no object ever created, updated, or stored anywhere.

One more distinction that flips the CRD mental model: the `metrics.k8s.io/v1beta1` schema (its `NodeMetrics`/`PodMetrics` types) is fixed by the Kubernetes project itself, not invented by whatever implements it. metrics-server is *one interchangeable implementation* of that fixed contract — the HPA controller only knows the group/version/schema, so a different implementation could sit behind the same `APIService` and nothing downstream would notice. A CRD has no equivalent notion of a swappable backend, because the schema **is** whatever the CRD's author defined, and apiserver+etcd is always the implementation.

---

## The Full Chain

```
HPA controller (inside kube-controller-manager)
  │  GET /apis/metrics.k8s.io/v1beta1/namespaces/<ns>/pods?labelSelector=...
  ▼
kube-apiserver — kube-aggregator matches the APIService, proxies the whole request
  ▼
metrics-server (a regular Deployment, not a DaemonSet)
  │  lists/watches Nodes via the normal API to learn every node's address, then
  │  makes its own direct outbound call to EACH node's kubelet — same
  │  /stats/summary endpoint, same port 10250, same TLS trust relationship
  │  described for the kubelet's own API surface
  ▼
kubelet (cAdvisor already running embedded in this same process — nothing
  extra to install here; it's compiled into the kubelet binary itself)
```

The direction worth correcting explicitly: it's **pull all the way down**, not push. HPA pulls from the apiserver; the apiserver's aggregation layer pulls (proxies to) metrics-server; metrics-server pulls from every kubelet on its own schedule; the kubelet's embedded cAdvisor already had the number sitting in memory, so that last hop is just a read, not a pull either. Nothing in this chain is ever pushed upward by a lower layer on its own initiative. metrics-server's own cache is in-memory only — no history, which is also why it can't back a dashboard the way a scrape-and-store system like Prometheus can.

A common real-world failure mode sits at the metrics-server → kubelet hop: metrics-server is making a direct HTTPS connection to each kubelet's serving certificate, and on self-managed clusters with self-signed kubelet certs (kubeadm, kind, minikube-style setups), that cert often isn't signed by a CA metrics-server trusts by default. That's the actual reason `--kubelet-insecure-tls` shows up in nearly every metrics-server install guide — it isn't a generic fix, it's specifically papering over that cert-trust gap at this exact hop.

---

## The Custom Metrics API: What Problem It Solves

> **Question:** `metrics.k8s.io` already gives HPA a working metrics path. Why does a second, structurally different API group exist instead of just teaching that one to return more kinds of numbers?

Because its schema has no room for more kinds of numbers. `PodMetrics` has exactly one shape: a `containers[]` array where each entry carries `usage.cpu` and `usage.memory` and nothing else. There's no field to put "http requests per second" or "queue depth" into — the type was written around two specific numbers, not an arbitrary one.

`custom.metrics.k8s.io` exists to be the opposite kind of schema: a generic envelope built around a name and a value, instead of fixed fields.

**Side by side, the divergence is visible directly in the response body.**

`metrics.k8s.io/v1beta1` (`PodMetrics`) — fixed, no metric identity of its own:

```json
{
  "kind": "PodMetrics",
  "containers": [
    { "name": "app", "usage": { "cpu": "5m", "memory": "10Mi" } }
  ]
}
```

`custom.metrics.k8s.io/v1beta2` (`MetricValueList`) — generic, the metric names itself:

```json
{
  "kind": "MetricValueList",
  "items": [
    {
      "describedObject": { "kind": "Pod", "name": "pod-x", "apiVersion": "v1" },
      "metricName": "http_requests_per_second",
      "value": "12500m"
    }
  ]
}
```

The difference is which side carries meaning. In `PodMetrics`, the meaning ("this is CPU," "this is memory") is baked into the field name — `usage.cpu` always means CPU, because the schema's author decided that at design time. In `MetricValueList`, the meaning is data, not schema: `metricName` is just a string, so the same type can carry `http_requests_per_second` today and `queue_depth` tomorrow without the API definition ever changing. What's still fixed is the *frame* around that string: every item is a value paired with a `describedObject` — some specific Kubernetes object the number is claimed to be about. That's the actual problem this API solves: giving an arbitrary metric a standard, machine-checkable way to say "this number belongs to this Pod," so HPA (or any other client) can consume it without knowing in advance what the number even is.

---

## A Tempting Shortcut: Pointing the Aggregation Layer at Prometheus Directly

A reasonable first instinct, once Prometheus is already in the cluster scraping this exact data: it already has `http_requests_per_second` sitting in its TSDB, so why not register an `APIService` for `custom.metrics.k8s.io/v1beta2` with `service:` pointing straight at Prometheus's own `Service`, and skip building anything new?

Recall from the aggregation-layer mechanics earlier: `kube-aggregator` doesn't inspect or validate what's behind an `APIService` at registration time — it just proxies the raw HTTP request to whatever `Service` the object names, and trusts whatever comes back. Nothing stops the `APIService` object itself from applying cleanly this way. It breaks the moment anything actually tries to use it:

- **Prometheus was never asked to speak the Kubernetes API's List/Get/Watch conventions.** kube-aggregator can proxy the bytes, but it can't make Prometheus's server understand a request like `GET /apis/custom.metrics.k8s.io/v1beta2/namespaces/default/pods/*/http_requests_per_second` — its HTTP server only understands its own query endpoint (`/api/v1/query`, taking a PromQL expression as a parameter), not this URL shape at all.
- **Even given a translated URL, the response shape is still wrong.** Prometheus's answer is a PromQL result — a list of `{metric labels, timestamp, value}` tuples — not a `MetricValueList`. There's no `describedObject`, no `metricName` field; `pod: "pod-x"` is just one label among however many the series carries. HPA's controller would try to unmarshal that body as a `MetricValueList` and fail outright, because the two JSON shapes have nothing in common beyond both being JSON.

The shortcut doesn't fail at registration — `kubectl apply` on the `APIService` succeeds either way, since nothing checks the backend's actual behavior up front. It fails downstream, silently from the aggregation layer's point of view, the first time anything tries to read from it. That gap — something has to speak the aggregation layer's expected HTTP contract *and* reshape PromQL's answer into a `MetricValueList` — is exactly the job Prometheus Adapter exists to do.

---

## Prometheus Adapter: Its Exact Role

Prometheus Adapter is a small extension apiserver — the same category of thing as metrics-server, built on the same `k8s.io/apiserver` library, running as its own Deployment with its own serving cert — whose entire job is closing that gap in both directions at once:

1. **Speaking the contract.** It implements the actual List/Get/Watch semantics `custom.metrics.k8s.io` requires, including the discovery call (`kubectl get --raw /apis/custom.metrics.k8s.io/v1beta2` lists every metric it currently knows how to serve) that lets a client discover what's available before asking for a specific one.
2. **Translating the request.** Given an incoming request naming a metric and a Kubernetes object selector, it consults a set of configured **rules** to decide which PromQL expression corresponds to that metric name, substitutes in the object's identity (namespace, label selector, resource type), and issues that query against Prometheus's ordinary `/api/v1/query` endpoint — the same endpoint any other Prometheus client would use.
3. **Translating the response.** It takes the PromQL vector result back and, for each series, builds one `MetricValueList` item: the series' relevant label (e.g. `pod`) becomes the `describedObject`, the configured metric name becomes `metricName`, and the scalar becomes `value`.

Nothing about Prometheus itself changes to make this work — from Prometheus's perspective, the adapter is just another client issuing ordinary queries on some schedule. All of the Kubernetes-API-shaped behavior lives entirely in the adapter process; Prometheus never needs to know this consumer is any different from a person poking around in its UI.

### A Config Example, Traced End to End

A representative adapter rule, in the shape of its `seriesQuery`-based config:

```yaml
rules:
  - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace: { resource: "namespace" }
        pod: { resource: "pod" }
    name:
      matches: "^(.*)_total$"
      as: "${1}_per_second"
    metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
```

Walking one request through it:

- HPA (via kube-apiserver) asks for `http_requests_per_second` on `Pods` matching some label selector, in namespace `default`.
- The adapter matches this against the `name.as` pattern (`http_requests_per_second` reverses to the underlying series `http_requests_total`) and the `resources` block (a `pod`-scoped metric).
- It fills in the `metricsQuery` template and issues, against Prometheus's own query API: `sum(rate(http_requests_total{namespace="default", pod=~"..."}[2m])) by (pod)` — the exact string a person would type into Prometheus's own UI to eyeball the same number for reference.
- Prometheus returns its native vector result (label set + scalar) for however many pods matched.
- The adapter reshapes each series into a `MetricValueList` item — `describedObject` from the `pod` label, `metricName: http_requests_per_second`, `value` from the scalar — and that's the body kube-apiserver hands back up the chain to HPA.

The `metricsQuery` template is worth sitting with for a moment: it isn't a fixed query, it's PromQL with placeholders the adapter fills in per request. That's what lets one rule serve the metric for *any* pod selector HPA asks about, rather than needing a separate rule per object.

---

## One Rule Per Metric, One Adapter Per Cluster

Every one of those rules lives in a single shared ConfigMap for the whole adapter Deployment — one metric-to-query mapping added by hand, cluster-wide, covering every team's every custom metric. There's no per-metric Kubernetes object to create, inspect with `kubectl get`, or scope with RBAC to a single team; changing what's scalable means editing one shared file and reloading one shared pod. It's fair to wonder whether this is the ceiling on the pattern, or whether the same translate-and-serve idea could be pushed further — one small declarative object per scaling target instead of one shared config file, reconciled by a controller, generalized past Prometheus to whatever backend a given target actually needs. That's a thread worth pulling on its own later.

---

## What This Covers So Far

This section now covers the resource-metric path (HPA scaling on CPU/memory via metrics-server) and the custom-metric path (HPA scaling on an arbitrary Prometheus-backed metric via Prometheus Adapter). It doesn't yet cover how KEDA layers on top of HPA to add scale-to-zero, how VerticalPodAutoscaler's evict-and-recreate model interacts (and conflicts) with HPA, or how Cluster Autoscaler provides the node capacity HPA's desired replica count assumes exists — see `kubernetes_scenarios.md`'s node-replacement scenario for that last piece in the meantime. Each of those is a candidate for its own section here.
