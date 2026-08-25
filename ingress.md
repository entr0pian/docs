# Ingress: Routing Rules and the Controllers That Implement Them

`Ingress` is a built-in API type (`networking.k8s.io/v1`) — not a CRD, ships with the API server itself. Like every object covered elsewhere in this repo, it's inert on its own: applying an `Ingress` just writes a record into etcd. Nothing routes traffic, provisions infrastructure, or does anything at all until a controller is watching it. `spec.ingressClassName` is what resolves *which* controller acts on a given `Ingress`, for clusters running more than one (a real scenario — e.g. both `nginx` and `alb` installed side by side).

This doc traces the two most common implementations end to end: the **AWS Load Balancer Controller** (an external, cloud-API-driven model) and **ingress-nginx** (an in-cluster, config-file-driven model). They arrive at the same API contract from structurally opposite directions.

---

## The Ingress Spec: Field Reference

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

### `pathType` Semantics

| `pathType` | Matching rule | Portable across controllers? |
|---|---|---|
| `Exact` | Full path must be character-identical. `/health` does **not** match `/health/live`. | Yes |
| `Prefix` | Path-*element*-wise match (split on `/`, compare element by element). `/v2` matches `/v2/orders`, but not `/v2extra` (naive string-prefix would wrongly match that). | Yes |
| `ImplementationSpecific` | Undefined by the Kubernetes API — entirely up to the controller. nginx-ingress treats it as a raw regex (PCRE-style), enabling capture groups. | **No** — meaning can silently change across controllers. |

`rewrite-target` (nginx-ingress-specific) rewrites the URI *before* proxying upstream, using a capture group from an `ImplementationSpecific` regex path — e.g. `rewrite-target: /$2` with path `/v1(/|$)(.*)` turns a request for `/v1/users/42` into `/users/42` by the time it reaches `api-v1-svc`, via an actual generated `rewrite ^/v1(/|$)(.*) /$2 break;` nginx directive. This decouples the backend's internal routes from whatever prefix it happens to be externally exposed under. Note the annotation is set once per `metadata`, so it applies to *every* path in that Ingress object — a real gotcha when only one path in the object actually has a capture group to fill.

---

## Implementation 1: AWS Load Balancer Controller

### Step by Step: What Gets Provisioned

1. The controller watches `Ingress` objects with `ingressClassName: alb`.
2. On seeing one, it calls the AWS API to create a real **ALB** — an actual resource outside the cluster, with its own ENI in your VPC.
3. It creates one **target group per backend `Service`** referenced across the Ingress's rules.
4. Each Ingress `rule` (host + path + backend) becomes an ALB **listener rule** — priority-ordered, matching on host-header/path-pattern, first match wins, forwarding to the corresponding target group.
5. Target registration happens in one of two modes, chosen via `alb.ingress.kubernetes.io/target-type`:
   - **`instance`** — targets are `NodeIP:NodePort` pairs. Requires the backend `Service` to carry a `NodePort`. Traffic still flows through kube-proxy's DNAT chain, and `externalTrafficPolicy` still governs cross-node masquerade/client-IP behavior, same as the plain `NodePort` case.
   - **`ip`** — targets are **Pod IPs directly**, sourced from `EndpointSlice` (already readiness-gated — a Pod failing its readiness probe is never registered). Requires the AWS VPC CNI, because a Pod IP must be a real, VPC-routable address for an external resource like the ALB to reach it directly — no NAT, no kube-proxy, no NodePort involved at all.

### Walking a Request (IP Mode)

A client requests `https://api.example.com/v1/users/42`:

1. TLS terminates at the ALB (cert from ACM, referenced via annotation — not a Kubernetes `Secret`, unlike nginx-ingress).
2. The ALB's listener rules evaluate host + path in priority order; the matching rule forwards to the `api-v1-svc` target group.
3. Within that target group, the ALB picks a target using its configured algorithm — **round robin by default**, with **least outstanding requests** available as an opt-in target-group attribute. (Ties back to the balancing-algorithm table in `services_and_load_balancing.md` — ALB gives you the "no load awareness" row unless you explicitly ask for the other one.)
4. The request goes straight to the chosen Pod's IP — no kube-proxy, no conntrack, no `NodePort` involved anywhere in this mode.

No proxy Pod runs in-cluster for this implementation at all — the "ingress controller" is purely a reconciler translating Kubernetes objects into AWS API calls.

---

## Implementation 2: ingress-nginx

### The Service It Still Needs

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

This is the *only* `LoadBalancer` Service that exists in the whole cluster under this model — every application behind it is exposed via its own `Ingress` object instead, never getting an LB of its own. Traffic reaching it follows exactly the `NodePort` mechanism traced in `services_and_load_balancing.md`: cloud NLB → `NodeIP:NodePort` → kube-proxy DNAT → an nginx Pod. `externalTrafficPolicy: Local` is close to mandatory here — `Cluster` would mean nginx sees `NodeA_IP` as the "client" for every request instead of the real one, breaking its own logging, rate limiting, and `X-Forwarded-For`.

### Two Different Sync Mechanisms, Not One

Once traffic reaches an nginx worker process, the controller (a Go binary running in the same container as nginx itself) keeps routing in sync via two entirely different paths, depending on what changed:

**Structural changes** (a new `Ingress` rule, a TLS Secret update, an annotation that changes an actual nginx directive) — the controller regenerates `nginx.conf` from a Go template and signals nginx to reload: `SIGHUP` to the master process, which parses the new config, spawns fresh worker processes, and lets the old ones gracefully drain in-flight connections before exiting.

**Endpoint changes** (a Pod becoming ready, a Deployment scaling, a rolling update) — these happen far more often than someone editing an `Ingress`, and reloading nginx on every single one would be disruptive at real scale. So this path skips `nginx.conf` and `SIGHUP` entirely:

1. The Go controller watches `EndpointSlice`.
2. On a change, it `POST`s the new backend list as JSON to a small **internal-only HTTP endpoint** the controller itself runs inside nginx (`127.0.0.1:10246/configuration/backends` — unreachable from outside the Pod).
3. A Lua handler receiving that POST writes the new list into a **`lua_shared_dict`** — a fixed-size memory region, `mmap`'d as genuinely shared across all of nginx's independent worker processes (which normally have no way to see each other's private memory).
4. On every actual client request, a `balancer_by_lua_block` — an OpenResty-provided hook for dynamic upstream selection — reads that same shared dict to pick a backend for *that specific request*.

Every worker sees the same live-updated backend list, with zero config-file involvement and zero dropped connections. This is why ingress-nginx is built on **OpenResty** (nginx + `lua-nginx-module`) rather than stock nginx — stock nginx has no mechanism for a worker process to react to anything at request time beyond what's already compiled into its config.

---

## Side by Side

| | AWS Load Balancer Controller | ingress-nginx |
|---|---|---|
| Where routing happens | An external AWS resource (ALB) | In-cluster Pods, running real nginx workers |
| Fronting infrastructure | Created dynamically, per Ingress | Static — one shared `LoadBalancer` Service, installed once |
| Structural-change propagation | `CreateRule`/`ModifyRule` AWS API calls | Regenerate `nginx.conf` + `SIGHUP` reload |
| Endpoint-change propagation | `RegisterTargets`/`DeregisterTargets` AWS API calls | Lua `POST` → `lua_shared_dict`, no reload |
| Backend target (IP mode / default) | Pod IPs directly (needs VPC CNI) or `NodeIP:NodePort` | Pod IPs directly, via Lua — never through kube-proxy |
| Default balancing algorithm | Round robin (least-outstanding-requests optional) | Round robin (configurable via `load-balance` annotation) |
| Protocol scope | HTTP/HTTPS/gRPC only (same Ingress limitation both share) | HTTP/HTTPS/gRPC only |

---

## What This Doc Doesn't Cover

- **The validating admission webhook.** ingress-nginx test-renders a new/changed Ingress against a running nginx instance before allowing it into etcd, catching bad annotation syntax or paths pre-emptively.
- **Canary / traffic-splitting annotations.** Weighted routing resolved from two separate Ingress objects into one nginx config.
- **Multiple Ingress objects sharing a class.** They all feed into the same generated `nginx.conf` — not isolated from each other; conflicting rules for the same host/path is a real failure mode.
- **Reload debouncing.** Whether rapid structural changes get batched before triggering `SIGHUP`.
- **`IngressClass`'s "is default" annotation**, and controller behavior when `ingressClassName` is omitted entirely.
- **The Gateway API** as the eventual, more expressive successor to `Ingress` — a separate resource family, not covered here at all.
