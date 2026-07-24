# Kubernetes Concepts — Quick Reference

## Deployment Strategy: RollingUpdate (default)

When no `strategy` is specified, Kubernetes uses `RollingUpdate` with these defaults:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%        # extra pods that can be created above desired count (rounds up)
    maxUnavailable: 25%  # pods that can be unavailable during rollout (rounds down)
```

### What this means at runtime

| replicas | maxSurge (25%, ceil) | maxUnavailable (25%, floor) | max pods alive | min pods alive |
|----------|----------------------|-----------------------------|----------------|----------------|
| 1        | 1                    | 0                           | 2              | 1              |
| 2        | 1                    | 0                           | 3              | 2              |
| 4        | 1                    | 1                           | 5              | 3              |
| 8        | 2                    | 2                           | 10             | 6              |

### Key behaviours
- Kubernetes creates new pods **before** terminating old ones when `maxUnavailable: 0`.
- A new pod is only counted as available once its **readiness probe** passes.
- Setting `maxSurge: 0` and `maxUnavailable: 1` gives a slower rollout with no extra capacity.
- Setting `maxSurge: 1` and `maxUnavailable: 0` guarantees zero downtime (requires headroom).

### Recreate strategy (alternative)
```yaml
strategy:
  type: Recreate  # terminates ALL old pods before creating new ones — causes downtime
```

---

## Probes

Kubernetes has three probe types, each serving a distinct purpose:

| Probe | Purpose | On failure |
|-------|---------|------------|
| `readinessProbe` | Is the pod ready to receive traffic? | Removed from Service endpoints — no traffic sent |
| `livenessProbe` | Is the pod still alive and not stuck? | Pod is **restarted** |
| `startupProbe` | Has the app finished starting up? | Pod is **restarted** if it never passes within the budget |

> `startupProbe` disables `liveness` and `readiness` checks until it succeeds, protecting slow-starting apps.

### Probe mechanisms

Each probe uses one of three mechanisms:

```yaml
# 1. HTTP GET — success if status code is 2xx or 3xx
httpGet:
  path: /healthz
  port: 8080
  httpHeaders:                  # optional
    - name: Custom-Header
      value: some-value

# 2. TCP socket — success if the port accepts a connection
tcpSocket:
  port: 5432

# 3. Exec — success if the command exits with code 0
exec:
  command:
    - cat
    - /tmp/ready
```

### Shared timing fields (all probe types)

```yaml
initialDelaySeconds: 0   # wait this long after container starts before first check
periodSeconds: 10        # how often to run the probe
timeoutSeconds: 1        # probe fails if no response within this time
successThreshold: 1      # consecutive successes needed to mark as healthy (readiness can be >1)
failureThreshold: 3      # consecutive failures before action is taken
```

> `successThreshold` must be `1` for `livenessProbe` and `startupProbe`.

### How they interact during a rollout

```
[container starts]
      │
      ▼
startupProbe (if defined)
  passes? ──yes──► livenessProbe + readinessProbe kick in
  fails within budget? ──► container restarted

readinessProbe passes ──► pod added to Service endpoints (traffic flows)
readinessProbe fails  ──► pod removed from endpoints (no traffic, not restarted)
livenessProbe fails   ──► container restarted (independent of readiness)
```

### Backend probe (current config)

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5   # first check after 5s
  periodSeconds: 10        # checked every 10s
  failureThreshold: 3      # unready after 30s of failures
# no livenessProbe or startupProbe defined
```

Since only a `readinessProbe` is configured, a stuck/deadlocked pod will **never be restarted automatically** — it will just stop receiving traffic.

---

## Graceful Shutdown

### The problem

During a rolling update, Kubernetes terminates the old pod by sending `SIGTERM` to the container process. Two things happen **at the same time** (not sequentially):

```
Pod marked Terminating
    │
    ├── 1. Pod removed from Service endpoints
    │       └── kube-proxy propagates the change to iptables/ipvs
    │           (this takes 1–5 seconds)
    │
    └── 2. SIGTERM sent to the container process immediately
```

If the application does not handle `SIGTERM`, the process exits right away. Because of the propagation delay in step 1, the load balancer may still route new requests to the pod for a few seconds **after** the process is already dead — causing connection errors.

Even if no new requests arrive, any **in-flight requests** being processed at the moment of SIGTERM are dropped mid-response.

### The solution: two layers

**Layer 1 — preStop hook (deployment)**

Add a `preStop` sleep to the pod spec. This delays `SIGTERM` by a few seconds, giving kube-proxy time to finish propagating the endpoint removal before the process is asked to stop.

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "5"]
```

**Layer 2 — graceful shutdown in the application**

Instead of `http.ListenAndServe` (which exits instantly on SIGTERM), use `http.Server.Shutdown()`. This:
1. Stops accepting **new** connections immediately
2. Waits for all **in-flight** requests to complete
3. Returns once the server is clean, or when the context deadline is reached

```go
srv := &http.Server{Addr: ":8080"}

// Start server in background goroutine
go srv.ListenAndServe()

// Wait for SIGTERM / SIGINT
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
<-quit

// Give in-flight requests up to 25s to finish.
// Stay under terminationGracePeriodSeconds (default 30s) to avoid SIGKILL.
ctx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
defer cancel()
srv.Shutdown(ctx)
```

### Timeline with both layers in place

```
t=0   Kubernetes marks pod Terminating
t=0   preStop sleep starts (no SIGTERM yet)
t=5   preStop finishes → kube-proxy has had time to drain the pod from endpoints
t=5   SIGTERM sent → app catches it, calls Shutdown()
t=5+  No new requests accepted; in-flight requests continue to be served
t=?   Last in-flight request completes → process exits cleanly
t=30  terminationGracePeriodSeconds deadline → kubelet sends SIGKILL (not reached in normal flow)
```

---

## External Secrets Operator (ESO) + AWS Secrets Manager

### How ESO bridges the two

```
AWS Secrets Manager          ESO controller              Kubernetes
─────────────────────        ───────────────             ──────────
secret: taskapp/dev/database ──► ExternalSecret ──────► Secret (in-cluster)
  {"password": "..."}             (CRD, watched)          POSTGRES_PASSWORD: ...
```

ESO runs as a controller inside the cluster. It watches `ExternalSecret` resources and periodically syncs their values from the upstream provider into native `Secret` objects. Pods consume the resulting `Secret` exactly as before — ESO is invisible to application code.

### Resource hierarchy

```
ClusterSecretStore   (cluster-scoped)
  └── defines HOW to reach AWS SM (region, auth method)

ExternalSecret       (namespace-scoped)
  └── defines WHAT to fetch (secret path, key mapping)
  └── references the ClusterSecretStore
  └── owns the resulting native Secret
```

### Authentication: static credentials (minikube)

In this cluster ESO authenticates to AWS using **static credentials** stored in a Kubernetes Secret named `aws-credentials` in the `external-secrets` namespace. The `ClusterSecretStore` references that secret directly:

```yaml
# argocd/cluster-secret-store.yaml
spec:
  provider:
    aws:
      service: SecretsManager
      region: eu-west-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-credentials
            namespace: external-secrets
            key: access-key-id
          secretAccessKeySecretRef:
            name: aws-credentials
            namespace: external-secrets
            key: secret-access-key
```

The `aws-credentials` Secret must be created manually before ArgoCD syncs the `ClusterSecretStore`:

```bash
kubectl create secret generic aws-credentials \
  --namespace external-secrets \
  --from-literal=access-key-id=<AWS_ACCESS_KEY_ID> \
  --from-literal=secret-access-key=<AWS_SECRET_ACCESS_KEY>
```

### ExternalSecret refresh and ownership

```yaml
spec:
  refreshInterval: 1h        # ESO re-reads from AWS SM every hour
  target:
    creationPolicy: Owner    # ESO owns the Secret; deleting ExternalSecret deletes the Secret
```

- `refreshInterval` controls how quickly rotation in AWS SM propagates into the cluster.
- `creationPolicy: Owner` means the generated `Secret` has an owner reference back to the `ExternalSecret`. It is garbage-collected automatically.

### Secret format convention

The secret stored in AWS SM is a JSON object:

```json
{"password": "the-actual-password"}
```

Path: `taskapp/dev/database` (configured in `helm-charts/database/values.yaml`).

The `ExternalSecret` extracts only the `password` property:

```yaml
data:
- secretKey: POSTGRES_PASSWORD   # key in the resulting K8s Secret
  remoteRef:
    key: taskapp/dev/database    # AWS SM secret path
    property: password           # JSON property to extract
```

Only `POSTGRES_PASSWORD` is secret. `POSTGRES_DB` and `POSTGRES_USER` are non-sensitive and live in a `ConfigMap`.

### ESO installation and ClusterSecretStore — managed by ArgoCD

ESO itself is installed via an ArgoCD `Application` (`argocd/external-secrets-app.yaml`), using the official Helm chart (version `0.14.4`). The `ClusterSecretStore` (`argocd/cluster-secret-store.yaml`) is also applied through ArgoCD.

Deployment order matters — the `aws-credentials` Secret must exist before the `ClusterSecretStore` syncs, otherwise ESO will report an auth error.

### SealedSecret for CloudNativePG

The CloudNativePG cluster (`postgres-cluster/`) uses a separate **SealedSecret** (`db-sealedsecret.yaml`) rather than ESO. This secret (`taskapp-database-cluster-secret`, type `kubernetes.io/basic-auth`) holds both `username` and `password` for the CNPG operator. It is encrypted with the cluster's Sealed Secrets controller key and committed to Git.

```
postgres-cluster/db-sealedsecret.yaml  →  Secret: taskapp-database-cluster-secret
                                           (consumed by the CNPG Cluster resource)
```

This is independent of the ESO-managed `POSTGRES_PASSWORD` secret used by the application pods.
