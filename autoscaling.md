# Autoscaling: HPA and the Metrics API

Covers how an application scales to handle traffic fluctuations. This entry starts with the HorizontalPodAutoscaler and the resource-metrics path that backs it — VerticalPodAutoscaler, custom/external metrics, KEDA, and Cluster Autoscaler are distinct enough topics to earn their own sections here later rather than a new file each.

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

## What This Covers So Far

This section is the resource-metric path only: HPA scaling on CPU/memory via metrics-server. It doesn't yet cover how a custom or external metric gets into this same chain (a Prometheus Adapter registering its own `APIService` for `custom.metrics.k8s.io`), how KEDA layers on top of HPA to add scale-to-zero, how VerticalPodAutoscaler's evict-and-recreate model interacts (and conflicts) with HPA, or how Cluster Autoscaler provides the node capacity HPA's desired replica count assumes exists — see `kubernetes_scenarios.md`'s node-replacement scenario for that last piece in the meantime. Each of those is a candidate for its own section here.
