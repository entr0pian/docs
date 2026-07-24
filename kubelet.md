# Kubelet & Node-Level Mechanics

The kubelet is the one control-plane-adjacent component that runs on every node and exposes its own HTTP(S) API, separate from `kube-apiserver`. This doc collects analyses of what that node-local API actually does and when the rest of the cluster talks to it directly instead of going through the central API server. Expect this file to grow new sections as more kubelet-adjacent questions come up, rather than spawning a new file per question.

---

## kubectl exec Internals: Namespace-Joining, Proxying, and the Kubelet's Own API

> **Question:** When you execute `kubectl exec -it <pod> -- bash`, are you logging into the pod?

### Short Answer

No. There's no login, no credential exchange with anything running inside the container, and no persistent shell process that was already there. `kubectl exec` creates a **brand-new process** and splices it into the namespaces of an already-running container. "Logging in" implies connecting to something that was waiting for you; what actually happens is closer to injecting a new, unrelated process into a box that already exists.

### What "Attaching" Actually Means: New Process, Existing Namespaces

The new `bash` doesn't ptrace-attach to PID 1 or take over its stdio. The container runtime forks a fresh process and calls `setns()` to join it into the target container's namespaces before exec-ing your shell. Inside that container's PID namespace, your shell shows up as a sibling process next to PID 1 (e.g. PID 7) — not as PID 1 itself.

Which namespaces actually get joined isn't uniform across a Pod, and this is the part people get wrong:

| Namespace | Scope |
|---|---|
| Network, IPC | **Pod-level** — shared by every container in the Pod |
| PID | **Per-container by default** — pod-wide only if `shareProcessNamespace: true` is set |
| Mount, UTS | Per-container |

The reason network/IPC are shared pod-wide at all is the pause (infra/sandbox) container — see `cadvisor_metrics.md` for why it exists and what it's holding open. `kubectl exec -c <container>` joins that specific container's mount+PID namespace, plus the Pod's shared network+IPC namespace inherited from the pause container. The new process's stdin/stdout/stderr aren't "wired to" anything pre-existing either — they're fresh pipes (or a pty, since `-it` requests one) created for this process alone, and those pipes are what get streamed back to your terminal.

### The Full Request Path

```
kubectl exec -it <pod> -- bash
  │
  ├─ 1. kubectl → kube-apiserver
  │     GET /api/v1/namespaces/<ns>/pods/<pod>/exec
  │         ?container=<c>&command=bash&stdin=true&stdout=true&stderr=true&tty=true
  │
  ├─ 2. apiserver authenticates the request and RBAC-checks the
  │     "pods/exec" subresource — this is the primary authorization gate
  │
  ├─ 3. apiserver looks up which node the Pod is scheduled on and
  │     proxies the request to that node's kubelet, using its own
  │     client credentials (the kubelet trusts the apiserver's CA)
  │
  ├─ 4. the connection upgrades to a multiplexed streaming protocol
  │     (SPDY historically, WebSocket in newer clusters) carrying
  │     stdin / stdout / stderr / resize / error as separate channels
  │     over one TCP connection
  │
  ├─ 5. kubelet's own HTTP server receives it at
  │     GET /exec/<namespace>/<pod>/<container>
  │
  ├─ 6. kubelet calls the container runtime's exec RPC (containerd/CRI-O),
  │     whose shim performs the fork + setns() + exec described above
  │
  └─ 7. stdio streams tunnel back up the same chain: runtime → kubelet
        → apiserver → kubectl → your terminal
```

The apiserver never touches the container directly — it's purely an authenticating router. All the namespace-joining and process creation happens on the node, driven by the kubelet talking to the container runtime.

This is the same shape `logging.md` already covers for a different endpoint: `kubectl logs` is `kubectl → apiserver → kubelet`, reading a file instead of spawning a process. `exec`, `attach`, and `portForward` are the odd ones out among the kubelet's endpoints precisely because they're long-lived bidirectional streams rather than a single request/response — which is exactly why this path needs a protocol *upgrade* instead of a normal HTTP proxy.

**One layer deeper — authorization doesn't fully stop at the apiserver's RBAC check.** A kubelet run with `--authorization-mode=Webhook` (the standard production setting) doesn't blindly trust every request the apiserver forwards to it. For each proxied request it can issue its own `SubjectAccessReview` call *back* to the apiserver, asking "is the original user actually allowed to do this?" — using the impersonated user identity the apiserver attached to the proxied call, not the apiserver's own service identity. So the trust chain isn't "apiserver checked once, kubelet blindly executes" — it's apiserver-authenticates-user → apiserver-authorizes-at-the-API-object-level → kubelet independently re-authorizes the same user against the same RBAC rules before actually touching the container. Two authorization checks, same policy source, different components asking.

### Kubelet's Exposed Endpoints (port 10250, HTTPS)

| Endpoint | Description |
|---|---|
| `GET /pods` | PodList — every Pod the kubelet knows about on this node, from in-memory state (this is what `logging.md`'s label lookup calls) |
| `GET /exec/{ns}/{pod}/{container}` | Streaming exec — backs `kubectl exec` |
| `GET /attach/{ns}/{pod}/{container}` | Streaming attach to a container's existing PID 1 stdio — backs `kubectl attach` |
| `GET /portForward/{ns}/{pod}` | Tunnels a local port into the Pod's network namespace — backs `kubectl port-forward` |
| `GET /stats/summary` | cAdvisor-derived resource usage; what `kubectl top`/metrics-server scrape per node |
| `GET /metrics/cadvisor` | Raw cAdvisor metrics — see `cadvisor_metrics.md` for the pause-container/aggregate label noise this produces |
| `GET /logs/` | Browse the kubelet's own log directory (not container logs — that's the file-tailing path `logging.md` describes) |
| `GET /healthz`, `/configz` | Kubelet's own health/config introspection |

### The Part That Should Feel Unsettling: Your Shell Doesn't Survive a Restart

`pod_lifecycle_and_restarts.md` already establishes that a container restart isn't an in-place resume — it's a fresh container instance, with fresh namespaces, fresh cgroups, a new PID 1. Put that together with what this section just established: your `exec` shell is a process living *inside* that specific container instance's PID namespace, not the Pod's.

So if the container you're exec'd into crashes and restarts while your shell is open, the namespace your shell was a member of doesn't get repaired or handed to the new instance — it's torn down along with the old container. Your session doesn't hang or prompt to reconnect; the process it depended on ceases to exist. There's no "re-attach" the way there might be for a detached tmux session — you have to `exec` again, fresh, into whatever new instance now exists.

A second, related tension worth sitting with: if a Pod sets `shareProcessNamespace: true`, an `exec` shell into *any one* container can see — and `kill` — processes belonging to every *other* container in that Pod. The isolation you'd assume between "different containers" from a separate-hosts mental model is opt-out at the Pod level, not a hard guarantee of the container abstraction itself.
