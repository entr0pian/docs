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

## LoadBalancer: A Cloud LB Provisioned on Top of NodePort

`type: LoadBalancer` doesn't replace the mechanisms above — it's built directly on top of both of them. Creating one still allocates a `ClusterIP` *and* a `NodePort` under the hood (visible as the `NodePort` column in `kubectl get svc`, unless explicitly disabled with `allocateLoadBalancerNodePorts: false`). The only genuinely new piece is a component that provisions a real external load balancer and wires its targets to that `NodePort`.

### Who Provisions It: the cloud-controller-manager

The **cloud-controller-manager's Service controller** watches for Service objects with `spec.type: LoadBalancer`, same watch-reconcile pattern as every other controller in this repo. On seeing one, it calls the cloud provider's API to create the real resource — an AWS NLB, a GCP external passthrough LB, etc. — configures its targets as `NodeIP:NodePort` across the cluster's nodes, then writes the result back onto the Service object's `status.loadBalancer.ingress` field once provisioning completes (that's what populates the `EXTERNAL-IP` column in `kubectl get svc`).

Once provisioned, traffic follows exactly the path already traced in the `NodePort` section above — the cloud LB is just what's now sending packets at `NodeIP:NodePort`, and `externalTrafficPolicy` still governs whether that forwarding can cross nodes (and lose client IP) or stays node-local.

### Why the Default LB Is L4, Not L7

This is a direct consequence of how `Service` is defined as an API object: `spec.ports` only carries a port number and a protocol (`TCP`/`UDP`) — no HTTP semantics, no concept of a host header or a path. The Service controller has nothing HTTP-aware to hand to the cloud API even if it wanted to; an L7 LB's API (listener rules matching host/path) needs information a `Service` object was never designed to carry. **This is the structural reason `Ingress` exists as its own resource type** rather than `Service` simply growing more fields — `Service` stays deliberately protocol-agnostic, and `Ingress` is the purpose-built surface for carrying L7 semantics down to an L7-capable LB.

The practical result, tying back to this doc's opening question: a bare `type: LoadBalancer` Service gives you the exact same **spread, not balance** ceiling as `ClusterIP`/`NodePort` — it's still just an L4 mechanism, one layer further out.

### When Spread Is Actually Fine

- **Non-HTTP protocols.** `Ingress` is HTTP/HTTPS-only by definition — a DNS server, an MQTT broker, a UDP game server, raw database access, or an SSH bastion has no L7 option to begin with.
- **TLS passthrough / mTLS.** An L4 LB forwards encrypted bytes untouched; the app terminates TLS itself. An L7 proxy terminating and re-encrypting would need the private key, breaking true end-to-end client-cert auth.
- **One connection *is* the whole unit of work.** A persistent streaming session where per-request rebalancing wouldn't do anything useful anyway.
- **Minimizing moving parts for a single exposed service**, trading away L7 features for not having to run an ingress controller at all.

### Example: a plain TCP service, spread is enough

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-external
  namespace: cache
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local   # preserve real client IP, avoid the cross-node masquerade hop
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
      protocol: TCP
```

Redis's wire protocol isn't HTTP, so there's nothing for an L7 proxy to route on — a bare L4 LB is the only option, not a compromise.

### Example: the one LoadBalancer Service in an nginx-ingress cluster

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local   # nginx needs the real client IP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/component: controller
  ports:
    - name: http
      port: 80
      targetPort: 80
    - name: https
      port: 443
      targetPort: 443
```

This is the pattern the earlier Ingress discussion converges on: exactly **one** `LoadBalancer` Service exists in the whole cluster, paid for once, fronting the shared nginx-ingress-controller Deployment. Every application behind it is exposed via an `Ingress` object instead — never getting an LB of its own — so an L4 hop is paid once, and L7 balance (real per-request routing across backend Pods) is what nginx itself provides for every app behind that single entry point.

### What This Section Doesn't Cover

- **IP-mode `LoadBalancer` Services.** Newer versions of the AWS Load Balancer Controller can also manage plain `type: LoadBalancer` Services directly in IP mode (bypassing `NodePort` entirely, mirroring the Ingress IP-mode case covered elsewhere in this repo) — not the default/in-tree path described above.
- **`status.loadBalancer.ingress` propagation and DNS.** How long provisioning actually takes, and how a hostname/IP there becomes something clients can resolve.
- **Cross-zone load balancing settings** and their cost implications on cloud NLBs.
- **UDP-specific LB behavior** — health checking and connection tracking differ from TCP.

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

## Ingress: Two Ways to Implement an L7-Aware Entity

`Ingress` is the concrete API object behind the "L7-aware entity" described above. Like every object covered in this doc, it's inert on its own: applying an `Ingress` just writes a record into etcd — a built-in type (`networking.k8s.io/v1`, not a CRD), but nothing routes traffic until a controller watches it. `spec.ingressClassName` resolves *which* controller acts on a given `Ingress`, for clusters running more than one (a real scenario — e.g. both `nginx` and `alb` installed side by side).

This section traces the two most common implementations end to end: the **AWS Load Balancer Controller** (an external, cloud-API-driven model) and **ingress-nginx** (an in-cluster, config-file-driven model). They arrive at the same API contract from structurally opposite directions.

### The Ingress Spec: Field Reference

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: prod
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-example-com-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: api-v1-svc
                port:
                  number: 80
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2-svc
                port:
                  number: 80
          - path: /health
            pathType: Exact
            backend:
              service:
                name: health-svc
                port:
                  number: 8080
  defaultBackend:
    service:
      name: default-404-svc
      port:
        number: 80
status:
  loadBalancer:
    ingress:
      - hostname: my-alb-1234567890.us-east-1.elb.amazonaws.com
```

| Field | Purpose |
|---|---|
| `spec.ingressClassName` | Resolves which controller acts on this object, when more than one is installed. |
| `spec.tls[].secretName` | A `kubernetes.io/tls` Secret (cert + key), in the **same namespace** as the Ingress. The controller watches Secrets too, for cert rotation. |
| `spec.rules[].host` | Matched against the HTTP `Host` header / TLS SNI. |
| `spec.rules[].http.paths[].path` + `.pathType` | The path-matching rule — see below. |
| `spec.rules[].http.paths[].backend.service` | The backend `Service` name + port this rule routes to. |
| `spec.defaultBackend` | Fallback for requests matching no rule at all (a host with no matching rule, not merely an unmatched path). |
| `metadata.annotations` | The escape hatch for everything the spec itself can't express — controller-specific, non-portable (e.g. `nginx.ingress.kubernetes.io/*`, `alb.ingress.kubernetes.io/*`). |
| `status.loadBalancer.ingress` | Written back by the controller once the real LB is provisioned — populates `kubectl get ingress`'s `ADDRESS` column. |

#### `pathType` Semantics

| `pathType` | Matching rule | Portable across controllers? |
|---|---|---|
| `Exact` | Full path must be character-identical. `/health` does **not** match `/health/live`. | Yes |
| `Prefix` | Path-*element*-wise match (split on `/`, compare element by element). `/v2` matches `/v2/orders`, but not `/v2extra` (naive string-prefix would wrongly match that). | Yes |
| `ImplementationSpecific` | Undefined by the Kubernetes API — entirely up to the controller. nginx-ingress treats it as a raw regex (PCRE-style), enabling capture groups. | **No** — meaning can silently change across controllers. |

`rewrite-target` (nginx-ingress-specific) rewrites the URI *before* proxying upstream, using a capture group from an `ImplementationSpecific` regex path — e.g. `rewrite-target: /$2` with path `/v1(/|$)(.*)` turns a request for `/v1/users/42` into `/users/42` by the time it reaches `api-v1-svc`, via an actual generated `rewrite ^/v1(/|$)(.*) /$2 break;` nginx directive. This decouples the backend's internal routes from whatever prefix it happens to be externally exposed under. Note the annotation is set once per `metadata`, so it applies to *every* path in that Ingress object — a real gotcha when only one path in the object actually has a capture group to fill.

### Implementation 1: AWS Load Balancer Controller

#### Step by Step: What Gets Provisioned

1. The controller watches `Ingress` objects with `ingressClassName: alb`.
2. On seeing one, it calls the AWS API to create a real **ALB** — an actual resource outside the cluster, with its own ENI in your VPC.
3. It creates one **target group per backend `Service`** referenced across the Ingress's rules.
4. Each Ingress `rule` (host + path + backend) becomes an ALB **listener rule** — priority-ordered, matching on host-header/path-pattern, first match wins, forwarding to the corresponding target group.
5. Target registration happens in one of two modes, chosen via `alb.ingress.kubernetes.io/target-type`:
   - **`instance`** — targets are `NodeIP:NodePort` pairs. Requires the backend `Service` to carry a `NodePort`. Traffic still flows through kube-proxy's DNAT chain, and `externalTrafficPolicy` still governs cross-node masquerade/client-IP behavior, same as the plain `NodePort` case above.
   - **`ip`** — targets are **Pod IPs directly**, sourced from `EndpointSlice` (already readiness-gated — a Pod failing its readiness probe is never registered). Requires the AWS VPC CNI, because a Pod IP must be a real, VPC-routable address for an external resource like the ALB to reach it directly — no NAT, no kube-proxy, no NodePort involved at all.

#### Walking a Request (IP Mode)

A client requests `https://api.example.com/v1/users/42`:

1. TLS terminates at the ALB (cert from ACM, referenced via annotation — not a Kubernetes `Secret`, unlike nginx-ingress).
2. The ALB's listener rules evaluate host + path in priority order; the matching rule forwards to the `api-v1-svc` target group.
3. Within that target group, the ALB picks a target using its configured algorithm — **round robin by default**, with **least outstanding requests** available as an opt-in target-group attribute. (Same ceiling as the table above — ALB gives you the "no load awareness" row unless you explicitly ask for the other one.)
4. The request goes straight to the chosen Pod's IP — no kube-proxy, no conntrack, no `NodePort` involved anywhere in this mode.

No proxy Pod runs in-cluster for this implementation at all — the "ingress controller" is purely a reconciler translating Kubernetes objects into AWS API calls.

### Implementation 2: ingress-nginx

#### The Service It Still Needs

Unlike the AWS controller, nginx-ingress does its proxying inside ordinary Pods running *in* the cluster — so before any `Ingress` object does anything, something has to already get external traffic to those Pods. That something is a plain `Service` of `type: LoadBalancer`, bundled as a **static manifest in the install bundle/Helm chart** — not created dynamically per Ingress the way the ALB is. This has to be true structurally: the controller process *is* what runs inside those Pods, so it can't create its own fronting Deployment or Service — neither exists yet until it's already running.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local   # nginx needs the real client IP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/component: controller
  ports:
    - name: http
      port: 80
      targetPort: 80
    - name: https
      port: 443
      targetPort: 443
```

This is the *only* `LoadBalancer` Service that exists in the whole cluster under this model — every application behind it is exposed via its own `Ingress` object instead, never getting an LB of its own. Traffic reaching it follows exactly the `NodePort` mechanism traced above: cloud NLB → `NodeIP:NodePort` → kube-proxy DNAT → an nginx Pod. `externalTrafficPolicy: Local` is close to mandatory here — `Cluster` would mean nginx sees `NodeA_IP` as the "client" for every request instead of the real one, breaking its own logging, rate limiting, and `X-Forwarded-For`.

#### Two Different Sync Mechanisms, Not One

Once traffic reaches an nginx worker process, the controller (a Go binary running in the same container as nginx itself) keeps routing in sync via two entirely different paths, depending on what changed:

**Structural changes** (a new `Ingress` rule, a TLS Secret update, an annotation that changes an actual nginx directive) — the controller regenerates `nginx.conf` from a Go template and signals nginx to reload: `SIGHUP` to the master process, which parses the new config, spawns fresh worker processes, and lets the old ones gracefully drain in-flight connections before exiting.

**Endpoint changes** (a Pod becoming ready, a Deployment scaling, a rolling update) — these happen far more often than someone editing an `Ingress`, and reloading nginx on every single one would be disruptive at real scale. So this path skips `nginx.conf` and `SIGHUP` entirely:

1. The Go controller watches `EndpointSlice`.
2. On a change, it `POST`s the new backend list as JSON to a small **internal-only HTTP endpoint** the controller itself runs inside nginx (`127.0.0.1:10246/configuration/backends` — unreachable from outside the Pod).
3. A Lua handler receiving that POST writes the new list into a **`lua_shared_dict`** — a fixed-size memory region, `mmap`'d as genuinely shared across all of nginx's independent worker processes (which normally have no way to see each other's private memory).
4. On every actual client request, a `balancer_by_lua_block` — an OpenResty-provided hook for dynamic upstream selection — reads that same shared dict to pick a backend for *that specific request*.

Every worker sees the same live-updated backend list, with zero config-file involvement and zero dropped connections. This is why ingress-nginx is built on **OpenResty** (nginx + `lua-nginx-module`) rather than stock nginx — stock nginx has no mechanism for a worker process to react to anything at request time beyond what's already compiled into its config.

### Side by Side

| | AWS Load Balancer Controller | ingress-nginx |
|---|---|---|
| Where routing happens | An external AWS resource (ALB) | In-cluster Pods, running real nginx workers |
| Fronting infrastructure | Created dynamically, per Ingress | Static — one shared `LoadBalancer` Service, installed once |
| Structural-change propagation | `CreateRule`/`ModifyRule` AWS API calls | Regenerate `nginx.conf` + `SIGHUP` reload |
| Endpoint-change propagation | `RegisterTargets`/`DeregisterTargets` AWS API calls | Lua `POST` → `lua_shared_dict`, no reload |
| Backend target (IP mode / default) | Pod IPs directly (needs VPC CNI) or `NodeIP:NodePort` | Pod IPs directly, via Lua — never through kube-proxy |
| Default balancing algorithm | Round robin (least-outstanding-requests optional) | Round robin (configurable via `load-balance` annotation) |
| Protocol scope | HTTP/HTTPS/gRPC only (same Ingress limitation both share) | HTTP/HTTPS/gRPC only |

### What This Section Doesn't Cover

- **The validating admission webhook.** ingress-nginx test-renders a new/changed Ingress against a running nginx instance before allowing it into etcd, catching bad annotation syntax or paths pre-emptively.
- **Canary / traffic-splitting annotations.** Weighted routing resolved from two separate Ingress objects into one nginx config.
- **Multiple Ingress objects sharing a class.** They all feed into the same generated `nginx.conf` — not isolated from each other; conflicting rules for the same host/path is a real failure mode.
- **Reload debouncing.** Whether rapid structural changes get batched before triggering `SIGHUP`.
- **`IngressClass`'s "is default" annotation**, and controller behavior when `ingressClassName` is omitted entirely.
- **The Gateway API** as the eventual, more expressive successor to `Ingress` — a separate resource family, not covered here at all.

---

## Takeaway

A `ClusterIP` Service's "load balancing" is real, but bounded by what an L4 mechanism can possibly offer: an independent, one-shot decision per TCP connection, cached by conntrack for that connection's life, with zero visibility into anything happening afterward. That's spread — useful, often good enough given enough independent callers, but never load-aware. Actual balance requires an L7-aware entity that decouples the client-facing connection from the backend-facing ones and makes an informed decision per request, using a live feedback signal (connection count, in-flight requests, latency) that only exists because something is watching requests complete. Even then, count-based balance and cost-based balance are different problems — the first is solved by default, the second only if you explicitly ask for it.
