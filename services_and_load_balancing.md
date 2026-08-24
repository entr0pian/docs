# Services & Load Balancing: Spread vs. Balance

Covers how a `ClusterIP` Service actually gets a packet to a Pod, what that mechanism can and cannot achieve in terms of load balancing, and how an L7-aware entity (Ingress controller, service mesh sidecar, ALB) closes the gap.

---

## Walking a Packet: Pod → Service → Pod, and Back

A Pod `curl`s a Service's DNS name. Six distinct mechanisms hand the packet off to each other before a byte reaches the target Pod — none of them is "the Service" doing anything, because the Service object itself is just a record sitting in etcd. It never runs, watches, or touches a packet.

### 1. DNS Resolution

CoreDNS resolves `myservice.myns.svc.cluster.local` to the Service's `ClusterIP` (e.g. `10.96.5.20`). At this point nothing below has happened yet — the `ClusterIP` isn't bound to any real interface anywhere in the cluster. It exists purely as a rule, not an address anyone's network stack actually owns.

### 2. Leaving the Pod

The client Pod sends a packet with `dst=10.96.5.20:port`. It leaves the Pod's network namespace over its veth pair and arrives at the other end of that pair, which sits in the **root network namespace of the Pod's own node**. From the host kernel's point of view, a packet arriving on an interface — not generated locally by a process in the root netns — enters the netfilter pipeline at `PREROUTING`, before any routing decision is made.

### 3. DNAT: ClusterIP → Pod IP

`kube-proxy` runs as a DaemonSet, one instance per node. Every instance independently watches Services and EndpointSlices and, ahead of time, programs an identical rule set into its own node's netfilter tables — nothing is computed at packet-arrival time. The chain structure: `KUBE-SERVICES` matches `dst=ClusterIP:port` and jumps to a per-Service chain (`KUBE-SVC-<hash>`), which holds one weighted rule per Pod in the Service's EndpointSlice (`KUBE-SEP-<hash>`), selected by random-probability matching. (The same `KUBE-SERVICES` jump is also hooked into `OUTPUT`, covering the other case — traffic generated directly by a process in the node's own root netns, e.g. a `hostNetwork` Pod or the kubelet.)

This selection is only made **once per connection**: the first packet (the SYN, for TCP) is what gets evaluated against the chain, picking one backend Pod IP. The DNAT rewrite (`dst: ClusterIP:port → PodIP:targetPort`) and the chosen backend are then recorded as a conntrack entry, keyed by the connection's 5-tuple. Every subsequent packet in that same connection matches the conntrack entry directly and skips rule evaluation entirely — one connection, one backend, for its entire life.

### 4. Getting to the Right Node

The rewritten destination is a Pod IP, which very likely lives on a different node than the one that just performed the DNAT. This part isn't kube-proxy's job, and it isn't done per-Pod — it's the CNI plugin's job, set up per-**node**. Each node is allocated a slice of the cluster's overall Pod CIDR when it joins (e.g. node A gets `10.244.1.0/24`, node B gets `10.244.2.0/24`), so every other node only needs one route per node — "the whole `10.244.1.0/24` block is reachable via node A" — instead of one route per Pod. How that route actually moves the packet depends on the plugin: some encapsulate (VXLAN, wrapping the packet in another one addressed to the destination node), others do native L3 routing (BGP-advertised routes, or cloud-provider VPC routes for CNIs that hand Pods real VPC IPs).

### 5. The Last Hop: Host → Pod

Arriving at the correct node only solves "which machine." That node may be running dozens of Pods, each in its own network namespace, each behind its own veth pair — the kernel still needs to know which local interface delivers this specific Pod IP. That's a per-**Pod** host route, created the moment the CNI plugin sets up the Pod's veth pair (e.g. `10.244.1.5 via veth1234`). Node-level routing (step 4) is per-node and set up once; this last-centimeter delivery is per-Pod and set up at Pod creation.

### 6. The Reply: Undoing the DNAT, for Free

The reply packet leaves the target Pod with `src=PodIP`, but the client is waiting for a response from the `ClusterIP` it originally sent to, not from some Pod IP it's never seen. No second rule handles this — conntrack does it automatically. Because the original DNAT was recorded as part of the connection's tracked state, conntrack recognizes the reply as belonging to that same connection and applies the *inverse* transformation before the packet continues: `src: PodIP → ClusterIP`. One DNAT rule, applied on the way in; conntrack reverses it for free on the way out.

### What This Section Doesn't Cover

- **Readiness gating.** The EndpointSlice controller doesn't just match Pods by label — it also gates on readiness. A Pod matching the selector with a failing readiness probe doesn't show up as an endpoint at all, even though it matches the selector.
- **IPVS and eBPF dataplanes.** Everything above is kube-proxy's default iptables mode. IPVS mode (hash tables, real scheduling algorithms) and eBPF-based dataplanes (e.g. Cilium replacing kube-proxy entirely) do the same job through different mechanisms.
- **External traffic.** NodePort and LoadBalancer Services take a different path onto the cluster, with `externalTrafficPolicy` changing source-IP-preservation behavior — not covered here.
- **Headless Services.** A Service with `clusterIP: None` skips this entire mechanism — DNS returns Pod IPs directly, with no ClusterIP and no kube-proxy involvement at all.

---

## NodePort: Walking a Packet, External Client → Pod

`NodePort` reuses every mechanism from the `ClusterIP` walk above — same `KUBE-SVC-<hash>`/`KUBE-SEP-<hash>` chains, same one-decision-per-connection DNAT, same conntrack. What's new is only the entry point, and one wrinkle that never shows up in the pure-internal case: **conntrack state lives on a single node**, and the backend DNAT happens to pick might not.

### The Reservation

Creating a `NodePort` Service allocates exactly **one** port number from a fixed range (default `30000–32767`) — not a range per node, one specific port. `kube-proxy` then programs a `KUBE-NODEPORTS` rule for that port on **every node in the cluster**, including nodes running zero Pods that match the Service's selector. Any node's IP on that port is a valid entry point, regardless of where the workload actually lives.

### Same-Node Case: No New Mechanism

An external client sends a packet to `NodeA_IP:NodePort`. It hits `PREROUTING` on Node A, matches the `KUBE-NODEPORTS` chain instead of `KUBE-SERVICES`, but jumps into the exact same per-Service chain as before. If the Pod DNAT selects happens to live on Node A itself, nothing else changes — reply comes back, Node A's conntrack reverses the DNAT for free, done.

### Cross-Node Case: Why Per-Node Conntrack Matters

Say the DNAT on Node A selects a Pod that lives on Node B. The packet's `dst` gets rewritten to that Pod's IP and the CNI routes it to Node B — but its `src` is still untouched: the real external client's IP.

If nothing else happened, the Pod on Node B would reply directly to that real client IP via ordinary routing, entirely bypassing Node A. The client would receive a reply from a `PodIP` it never sent anything to, instead of from `NodeA_IP` where it believes the connection lives — indistinguishable from a spoofed packet, and dropped.

Node A is the only place the conntrack entry for this connection exists, so the reply *must* be forced back through it. `kube-proxy` does this with an extra step beyond DNAT: it also marks the packet for **masquerade** (`KUBE-MARK-MASQ`, applied via a `MASQUERADE` rule in `KUBE-POSTROUTING`), rewriting `src: ClientIP → NodeA_IP` before handing the packet to the CNI. Both halves — the DNAT to the Pod and the SNAT to Node A's own IP — are recorded together in Node A's conntrack entry.

Now the Pod on Node B believes *Node A* is the client, and replies there via plain routing (no CNI trickery needed for the return — Node A is just another routable node). Node A's conntrack recognizes the reply and reverses both transformations in one shot before the final packet goes out to the real client.

### `externalTrafficPolicy: Cluster` (default)

Any node may forward to any Pod cluster-wide, masquerading whenever the pick lands on a different node. Consequence: **the Pod never sees the real client IP** — it sees `NodeA_IP` instead, because that's who it thinks sent the request. The upside is even connection spread regardless of how Pods happen to be placed across nodes.

### `externalTrafficPolicy: Local`

`kube-proxy` refuses to select a cross-node backend at all — a node only forwards to Pods running locally on itself. No masquerade ever happens, so the real client IP survives end to end.

This requires the external load balancer to stop sending traffic to nodes with zero local ready Pods, which is what the auto-allocated `healthCheckNodePort` is for — a lightweight endpoint the cloud LB polls per node to learn its local ready-endpoint count, removing empty nodes from rotation.

The real cost: cloud load balancers distribute evenly across **healthy nodes**, not per-Pod. Two Pods on Node A and one Pod on Node B still get roughly equal traffic *per node*, meaning Node B's single Pod absorbs what Node A splits two ways. `Local` trades correctness (real client IP) for a new manual obligation: keeping Pods evenly spread across nodes yourself (anti-affinity / topology spread constraints), or eating silent hot-spotting.

| | `Cluster` (default) | `Local` |
|---|---|---|
| Cross-node forwarding | Allowed | Never — local Pods only |
| Masquerade / SNAT | Applied when crossing nodes | Never needed |
| Client IP seen by Pod | `NodeA_IP` (lost) | Real client IP (preserved) |
| Traffic distribution | Even across all Pods cluster-wide | Even across healthy *nodes*, not Pods — needs even Pod spread |
| Extra machinery | None | `healthCheckNodePort` |

### Where This Matters in Practice

`Local` is close to mandatory for anything internet-facing where source IP is meaningful — most notably the `LoadBalancer` Service fronting an nginx-ingress-controller Deployment, since nginx needs the real client IP for logging, IP-based rate limiting, and populating `X-Forwarded-For` correctly for the app behind it. `Cluster` would hand nginx `NodeA_IP` as "the client" for every request.

This entire mechanism is also worth bounding: it's irrelevant to the AWS Load Balancer Controller's IP-mode target groups, covered elsewhere in this repo — no kube-proxy, no NodePort, no conntrack involved there, since the ALB talks directly to Pod IPs. `externalTrafficPolicy` only governs NodePort-backed paths: ALB *instance* mode, bare-metal `MetalLB`, and nginx-ingress's own fronting Service.

### What This Section Doesn't Cover

- **`LoadBalancer` Service type and the cloud-controller-manager.** Which component actually watches for `type: LoadBalancer` and calls the cloud provider's API to provision the real external LB, and how it wires that LB's targets back to the NodePort mechanism above.
- **IPVS-mode differences.** Whether the masquerade requirement changes under IPVS instead of iptables.
- **`healthCheckNodePort` protocol details.** The exact request/response shape the cloud LB polls.
- **Session affinity interaction.** How `sessionAffinity: ClientIP` behaves once `externalTrafficPolicy: Local` is already pinning by node.
- **Dual-stack / IPv6 NodePort behavior.**

---

## Can a ClusterIP Service Ensure Load Balancing for TCP Traffic?

> **Question:** Can a Service of type `ClusterIP` ensure load balancing for TCP traffic?

**Short answer: it can only ever produce statistical *spread* across connections, never true load-aware *balancing* — and that ceiling isn't a missing feature, it's a structural consequence of operating at L4.**

### How a ClusterIP Actually Gets a Request to a Pod

(Full hop-by-hop trace: see "Walking a Packet: Pod → Service → Pod, and Back" above.)

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
