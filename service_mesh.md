# Service Mesh: Istio

A service mesh adds traffic management, security, and observability at the
network layer without touching application code, by putting a proxy in
front of every workload instead of relying on the app to speak TLS,
retries, or fine-grained routing itself. Istio is the concrete instance
covered here: a control plane (`istiod`) and a data plane (an Envoy proxy
injected as a sidecar into every Pod). This entry covers the control
plane/data plane split, how Istio's load balancing actually differs from
`kube-proxy`'s (`services_and_load_balancing.md`), a full end-to-end trace
of one proxied request, and canary/weighted-routing via `VirtualService`
and `DestinationRule`.

---

## Architecture: Control Plane and Data Plane

```
istiod (control plane, one Deployment for the whole mesh)
  │  watches, via the normal Kubernetes API: Services, Endpoints/
  │  EndpointSlices, Pods (for their labels), and Istio's own CRDs
  │  (VirtualService, DestinationRule, Gateway, ...)
  │
  │  translates all of that into Envoy's xDS config and pushes it over a
  │  single gRPC stream per proxy (xDS/ADS — Aggregated Discovery Service):
  │    CDS — Cluster Discovery   (what upstream "clusters" exist)
  │    EDS — Endpoint Discovery  (which Pod IPs back each cluster)
  │    LDS — Listener Discovery  (what ports/protocols this proxy accepts on)
  │    RDS — Route Discovery     (which cluster a request matches, incl. weights)
  ▼
Envoy sidecar (data plane, one per Pod)
  │  holds a local, continuously-updated snapshot of that xDS config
  │  every routing and load-balancing decision is made locally, entirely
  │  from that cached snapshot — no call back to istiod per request
  ▼
App container — same Pod, same network namespace, unaware any of this exists
```

Two mechanisms make a Pod part of the mesh at all, both driven by a
mutating admission webhook `istiod` registers:

- **A sidecar container** running Envoy, added to the Pod spec at creation
  time, alongside the app container for the Pod's whole lifetime.
- **An init container** (`istio-init`) that runs once, before the app or
  Envoy start, and writes the `iptables` rules — scoped to that Pod's own
  network namespace only — that transparently redirect traffic into Envoy.
  The exact rules and why they don't NAT the same way `kube-proxy` does are
  traced in the end-to-end section below.

Nothing here modifies the Kubernetes `Service` or `Endpoints`/
`EndpointSlice` objects themselves — from `kube-proxy`'s point of view nothing
in the cluster changed. Istio's entire mechanism sits on top of the plain
Kubernetes networking model, not in place of it.

---

## Load Balancing: The Least-Request Algorithm

`kube-proxy` spreads traffic with one fixed mechanism (a semi-random
`iptables`/IPVS rule, no notion of "how busy is this target"). Istio's
answer to "smarter than that" is a *choice* of algorithms Envoy implements
— the interesting one being `LEAST_REQUEST`, set explicitly via a
`DestinationRule`, since Istio's actual default is still plain round robin:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: helloworld
spec:
  host: helloworld
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST
```

The key thing to get right about this algorithm is *where the "busy" number
comes from*, because it's easy to assume it's centrally tracked given that
`istiod` is a central component. It isn't:

- `istiod` never pushes load data down at all. What it pushes via EDS is
  pure **membership** — which Pod IPs currently back a given cluster, and
  whether they're healthy. No request counts, no load figures, anywhere in
  that feed.
- Envoy's actual selection is "power of two choices" (P2C): pick two random
  endpoints from its local EDS-provided list, compare **its own** count of
  currently in-flight requests to each of those two, send to the lighter
  one.
- "Its own count" is the load-bearing detail: a given Envoy sidecar only
  knows about requests *it personally* sent that haven't completed yet. It
  has no visibility into how much traffic other sidecars elsewhere in the
  mesh are simultaneously sending that same target Pod. There is no
  component anywhere — not `istiod`, not any Envoy — that ever computes a
  true global "Pod X currently has N in-flight requests across the whole
  mesh" figure.

This is why it's `LEAST_REQUEST` and not `LEAST_LOAD`: it's a statistical
approximation that tends to even things out as enough independent,
locally-informed P2C decisions accumulate across the mesh, not an exact
global optimum. Separately, each Envoy also runs outlier detection —
ejecting a Pod from its own candidate list if it's erroring or slow — which
prunes *which* Pods are eligible, a different mechanism from picking
*among* eligible ones.

---

## Walking a Request End to End: Pod → Sidecar → Sidecar → Pod

Pod A's app calls `serviceA:port`. Pod B is one of the Pods backing it.

1. **DNS resolves as normal.** `serviceA` resolves to its Kubernetes
   ClusterIP — Istio hasn't touched DNS or the `Service` object at all.
2. **`iptables` inside Pod A's own network namespace intercepts the
   outbound connection.** The rule Istio's init container wrote uses the
   `REDIRECT` target — a real DNAT, using the same `conntrack` primitive
   `kube-proxy` uses, just scoped to this one Pod's namespace instead of
   node-wide. It rewrites the destination to `127.0.0.1:15001`, Envoy A's
   outbound listener.
3. **Because `conntrack` is tracking that translation, it's reversible per
   connection.** Envoy recovers the pre-NAT destination via
   `SO_ORIGINAL_DST` (so it knows "this was headed to serviceA"), and any
   traffic flowing back through this same connection gets automatically
   un-NAT'd — Pod A's app socket keeps seeing traffic as if it's still
   talking directly to `serviceA:port`, for the life of that connection.
4. **Envoy A terminates that connection** — it does not forward packets —
   matches the recovered destination against its CDS clusters, picks a
   target Pod IP from the matching cluster's EDS list via its LB algorithm,
   and **opens a brand-new, separate connection** to Pod B — typically to
   Pod B's Envoy on port 15006 if mTLS is enforced. This hop is not NAT of
   any kind; it's a fresh dial Envoy A makes itself, wrapping the traffic
   in mTLS if the mesh's `PeerAuthentication` policy requires it.
5. **`iptables` inside Pod B's own namespace intercepts the inbound
   connection** the same way — another local `REDIRECT`/`conntrack` rule,
   this time on the `PREROUTING` side, sending it to `127.0.0.1:15006`,
   Envoy B's inbound listener.
6. **Envoy B terminates that connection** (decrypting mTLS if it was used)
   and **opens yet another new, local connection** to
   `127.0.0.1:<app-port>` — handing the request to the app container. This
   has to be a real re-origination, not a NAT passthrough, because Envoy B
   needs to actually process the traffic (decrypt, apply policy) before the
   app sees it.
7. **The app container sees a plain connection from `127.0.0.1`.** It has
   no visibility into serviceA, mTLS, or either Envoy — it was never going
   to see the real client identity under `kube-proxy` either.
8. **The response flows back over the same two already-established
   connections** — Envoy B → Envoy A, then Envoy A → Pod A's app socket —
   with `conntrack` maintaining the illusion from step 3 the whole way.

The one-sentence version: two independently NAT'd local hops (app↔local
Envoy, on each end, via the same `conntrack` trick `kube-proxy` uses) bookend
one genuinely proxied hop in the middle (Envoy↔Envoy, a real new connection,
no NAT at all). `kube-proxy` only ever does the first kind of thing;
Istio's actual cross-Pod hop is a different mechanism entirely.

---

## Canary Deployments: VirtualService and DestinationRule

### The Resources

A canary split needs three things: a Kubernetes `Service` that doesn't
distinguish versions, two sets of Pods that do (via a label), and the two
Istio CRDs that encode the split itself.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: helloworld
spec:
  selector:
    app: helloworld        # matches both versions — this Service alone
                            # cannot tell them apart
  ports:
    - port: 9080
      targetPort: 9080
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld
spec:
  hosts:
    - helloworld
  http:
    - route:
        - destination:
            host: helloworld
            subset: v1
          weight: 90
        - destination:
            host: helloworld
            subset: v2
          weight: 10
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: helloworld
spec:
  host: helloworld
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

Both `helloworld-v1` and `helloworld-v2` Deployments carry
`app: helloworld` (so the plain Service selects both) plus a distinguishing
`version: v1` / `version: v2` label the Service's own selector never looks
at.

### Field Reference

| Resource | Field | Meaning |
|---|---|---|
| `VirtualService` | `spec.hosts` | Which Service this routing rule applies to — the same short name (or FQDN) any plain client would resolve. |
| `VirtualService` | `http[].route[].destination.host` | Restates the target Service (matters once a `VirtualService` has multiple `http` match blocks routing to different hosts). |
| `VirtualService` | `http[].route[].destination.subset` | A *name*, not a label selector — meaningless on its own; resolved by looking it up in a `DestinationRule` for the same `host`. |
| `VirtualService` | `http[].route[].weight` | Percentage of matched traffic sent to this destination; all weights in one `route[]` block must sum to 100. |
| `DestinationRule` | `spec.host` | Which Service this rule defines subsets for. |
| `DestinationRule` | `spec.subsets[].name` | The name a `VirtualService` references via `subset:`. |
| `DestinationRule` | `spec.subsets[].labels` | The actual Pod label selector a subset name expands to. |

**There is no object reference between the two CRs** — no name, no UID,
nothing the API server validates at admission time. They're linked purely
by both declaring the same `host: helloworld` string. Typo the
`DestinationRule`'s `host`, or forget to apply it, and `subset: v1` in the
`VirtualService` silently resolves to nothing — there's no admission-time
error for that mismatch.

### How the 90/10 Split Actually Gets Enforced

The plain Kubernetes `Endpoints`/`EndpointSlice` object behind `helloworld`
has both v1 and v2 Pod IPs mixed together with no version information at
all — that object is completely unaware canary routing exists.

The split is entirely `istiod`'s translation layer, materialized as
**separate Envoy clusters, one per subset**, not as a runtime filter over
one shared pool:

```
outbound|9080|v1|helloworld.default.svc.cluster.local   ← its own EDS list, v1 Pods only
outbound|9080|v2|helloworld.default.svc.cluster.local   ← its own EDS list, v2 Pods only
```

`istiod` builds each cluster's EDS endpoint list by cross-referencing the
plain `EndpointSlice` (which only has Pod IPs) against the Pod objects it's
separately watching (which carry the `version` label) — the "additional
info" needed to tell v1 and v2 apart has to come from that cross-reference,
since the `EndpointSlice` alone can't provide it.

The `VirtualService`'s 90/10 becomes a `weighted_clusters` entry in Envoy's
route config (RDS), naming those two cluster names directly with those
weights. So the weighting is applied exactly once, at the routing decision
in the client-side Envoy, over two pools that were already fully separated
upstream by `istiod` — not a percentage Envoy calculates per-request against
a shared, labeled bucket of endpoints.

---

## What This Covers So Far

This covers the control plane/data plane split and the exact xDS resources
`istiod` pushes (CDS/EDS/LDS/RDS); the `LEAST_REQUEST` load-balancing
algorithm and why it's locally, not centrally, computed; a full trace of
one request's two NAT'd local hops plus the one genuinely proxied
Envoy-to-Envoy hop in between; and canary/weighted routing via
`VirtualService`/`DestinationRule`, including that the two CRs are linked
only by a matching `host` string and that the 90/10 split resolves to
separate per-subset Envoy clusters, not a runtime filter.

Not yet covered: mTLS specifics (`PeerAuthentication`/`AuthorizationPolicy`,
Istio's built-in CA and workload cert rotation — flagged but not traced
here); the `Gateway` CRD and ingress-gateway architecture (Istio's own
answer to what `services_and_load_balancing.md` covers via the AWS Load
Balancer Controller / ingress-nginx); other traffic-management primitives a
`DestinationRule`'s `trafficPolicy` also controls — retries, circuit
breaking (`outlierDetection` in more depth), connection pool limits;
header-based or otherwise sticky canary routing as an alternative to
blind-percentage `weight`; and Istio's newer "ambient mode" data plane,
which removes the per-Pod sidecar entirely in favor of a per-node proxy —
a fundamentally different answer to the same control-plane/data-plane
split covered here.
