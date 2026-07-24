# Services & Load Balancing: Spread vs. Balance

Covers how a `ClusterIP` Service actually gets a packet to a Pod, what that mechanism can and cannot achieve in terms of load balancing, and how an L7-aware entity (Ingress controller, service mesh sidecar, ALB) closes the gap.

---

## Can a ClusterIP Service Ensure Load Balancing for TCP Traffic?

> **Question:** Can a Service of type `ClusterIP` ensure load balancing for TCP traffic?

**Short answer: it can only ever produce statistical *spread* across connections, never true load-aware *balancing* — and that ceiling isn't a missing feature, it's a structural consequence of operating at L4.**

### How a ClusterIP Actually Gets a Request to a Pod

There is no load-balancer *process* in this picture — no pod, no proxy sitting in the traffic path. `kube-proxy` runs on every node, watches Endpoints/EndpointSlice for the Service, and programs identical netfilter (iptables) or IPVS rules into every node's own kernel. `ClusterIP` isn't bound to a real interface anywhere — it exists only as a DNAT rule, intercepted wherever a packet happens to be processed, almost always the node the *client* pod is running on. The decision is made independently, locally, per node, with no shared state and no global view of current load.

Crucially, that decision is only made **once per connection**: rule evaluation happens on the very first packet of a flow — the SYN — which performs the DNAT (`ClusterIP:port` → chosen `PodIP:targetPort`) and caches the result as a conntrack entry, keyed by the connection's 5-tuple. Every later packet in that connection matches the conntrack table and gets the same translation automatically; the NAT rules are never re-evaluated. One TCP connection, one backend, for its entire life — not by policy choice, but because that's how the kernel's NAT/conntrack machinery works for any stateful protocol.

This is worth connecting back to the propagation-delay problem already documented in `concepts.md`'s Graceful Shutdown section: removing a Pod from Endpoints only ever stops *new* connections from choosing it — the 1–5 second `kube-proxy` propagation window matters for that reason. It says nothing about connections that were already established before removal. Those keep flowing to the terminating Pod, fully unaffected by the Endpoints change, for as long as conntrack holds the entry — which, in practice, is until the Pod is actually killed by `SIGKILL` or exits cleanly. So a Pod being "drained" from a load-balancing perspective and a Pod actually being empty of traffic are two different moments, sometimes far apart in time.

### What This Can Achieve: Spread, Not Balance

| Mode | Selection mechanism | Aware of backend load? |
|---|---|---|
| iptables (default) | Chained probabilistic rules — independent random choice per new connection | No |
| IPVS | Real scheduling algorithms (`rr`, `wrr`, `lc`) | Only connection *count*, and only at new-connection time |

Even in the best case, this is a ceiling imposed by L4 itself: a Service has no visibility into request boundaries inside a TCP byte stream, so it can only ever balance *between* connections, never *within* one. That interacts badly with how HTTP actually behaves in practice — virtually every HTTP client defaults to keep-alive/connection pooling specifically to *avoid* opening new connections, which is the opposite of what you'd need for connection-level spread to do anything useful. A sequential client reusing one pooled connection sends 100% of its traffic to whichever single backend that connection landed on; a concurrent client only opens as many connections as its pool ceiling allows, then reuses those.

In real service-to-service traffic this is often "good enough" anyway, because the calling service usually has many independent replicas, each making its own independent connection-selection roll — but that argument leans on two assumptions that quietly fail at the edges: enough independent callers for the randomness to actually average out (a handful of callers can land skewed by pure chance, the same way three coin flips can land three-for-three), and roughly uniform cost per request (spreading connection *count* evenly says nothing about compute *cost* if some requests are far more expensive than others).

`sessionAffinity: ClientIP` is worth naming here too — it doesn't add new capability, it just leans further into the same mechanism, pinning even *new* connections from a given client IP to the same backend within a timeout window. It's the same axis, deliberately extended rather than fought.

---

## The Proper Way: An L7-Aware Entity

Getting past the connection-level ceiling requires something that actually terminates the client's connection and understands the protocol riding on top of it — an Ingress controller, a service mesh sidecar proxy, or an L7 load balancer like an AWS ALB (not an NLB — that's still L4, and inherits the exact same per-flow pinning as `ClusterIP`).

The key structural change is that these have **two independent connection layers** instead of one continuous pipe:

- **Downstream (client-facing):** the client opens one TCP connection to the proxy. Multiple HTTP requests arrive over it — sequentially for HTTP/1.1 keep-alive, or as concurrently multiplexed streams for HTTP/2.
- **Upstream (backend-facing):** the proxy keeps its *own* pool of persistent connections to each backend Pod, entirely decoupled from whatever the client is doing.

Because these layers are decoupled, the proxy makes a fresh routing decision **per request**, not per connection — two requests arriving back-to-back on the same client connection can be forwarded to two completely different backends. This is exactly what fixes the gRPC/HTTP2 case discussed earlier: many multiplexed logical streams inside one client connection can be fanned out across many different backend Pods simultaneously, something a `ClusterIP` is structurally incapable of, no matter how the traffic is shaped.

This is not an assumption the proxy makes about your application being stateless — it's simply the proxy's unconditional default behavior, and that default is what *requires* backends to be stateless (or externalize state to something shared) unless you explicitly opt out. The opt-out is **session affinity/sticky sessions** — a cookie the proxy injects, source-IP hashing, or a header — which deliberately re-introduces the same connection-pinning behavior `ClusterIP` gives you by default, except now as a conscious trade-off rather than an accident of L4.

---

## How L7 Entities Actually Achieve Balance

The algorithms below are what turn "spread" into "balance" — and all of them share one requirement: a live, continuously-updated signal per backend, fed by a **completion event** ("this request just finished") that only exists because the proxy can see individual requests end.

| Algorithm | How it decides | Load-aware? |
|---|---|---|
| Round robin / weighted round robin | Fixed cyclic order (weighted by declared capacity) | No — blind to current backend state |
| Least connections | Route to whichever backend has fewest active *connections* right now | Coarse — connection count as a proxy for load |
| Least outstanding requests | Route to whichever backend has fewest in-flight *requests* right now | Better — request-level, not connection-level |
| Power of two choices (P2C) | Pick 2 backends at random, send to whichever has fewer outstanding requests | Yes, cheaply — a well-known result (Mitzenmacher) that this gets exponentially better balance than pure random, without tracking full global state. Used by Envoy and Linkerd by default. |
| Latency/EWMA-weighted | Skew traffic away from backends with rising response-time trends | Indirect — slow responses often mean load, but can also mean a struggling downstream dependency |

The reason none of this is available at L4: every one of these algorithms needs to know when a unit of work finishes so it can update its per-backend counters before the next decision. Conntrack has no concept of "a request finished" — it only ever sees "connection established" and "connection closed," and by the time a connection closes, whatever load-balancing decision it made is long over. L7 proxies can hook this because they parse the protocol and see request boundaries directly; that visibility is the actual mechanism behind "balance," not just a design preference.

One gap survives even the best of these algorithms: least-outstanding-requests and P2C balance the *number* of things in flight per backend, not the actual compute cost of each one. If request cost varies widely, a backend can be holding fewer, more expensive requests and still be the most overloaded Pod in the fleet — genuine load-based balancing (CPU- or latency-driven) requires an explicit signal for that, and most proxies don't do it by default.

---

## Takeaway

A `ClusterIP` Service's "load balancing" is real, but bounded by what an L4 mechanism can possibly offer: an independent, one-shot decision per TCP connection, cached by conntrack for that connection's life, with zero visibility into anything happening afterward. That's spread — useful, often good enough given enough independent callers, but never load-aware. Actual balance requires an L7-aware entity that decouples the client-facing connection from the backend-facing ones and makes an informed decision per request, using a live feedback signal (connection count, in-flight requests, latency) that only exists because something is watching requests complete. Even then, count-based balance and cost-based balance are different problems — the first is solved by default, the second only if you explicitly ask for it.
