# Application Logging: Collection, stdout, and the Risk of Loss

> **Question:** How should application logs be collected, and is there a risk of losing logs?

---

## Where stdout Actually Goes

An application writing to stdout/stderr never talks to Kubernetes directly — it's writing to a file descriptor the container runtime set up. The container runtime (containerd, CRI-O) redirects that stream to a log file on the **node's local disk**, one file per container.

The file lives at:

```
/var/log/pods/<namespace>_<pod-name>_<pod-uid>/<container-name>/<restart-count>.log
```

with a friendlier symlink at:

```
/var/log/containers/<pod-name>_<namespace>_<container-name>-<container-id>.log
```

This is the core fact everything else builds on: **stdout isn't ephemeral by default — it's already being written to disk, with metadata (namespace, pod, container, restart count) baked into the file's own name.** No agent is required for this part; the kubelet does it automatically for every container.

What *is* ephemeral is the file's lifetime: it's node-local, and the kubelet rotates it (fixed max size, fixed number of rotated files kept per container) independent of anything the application does.

## What `kubectl logs` Actually Does

`kubectl logs <pod>` is a live tail of that file, nothing more. The path is:

```
kubectl → kube-apiserver → kubelet (on the node running the pod) → reads the log file → streams it back
```

`kubectl logs --previous` works because the kubelet keeps the *immediately preceding* terminated container's log file around (that's what the `<restart-count>.log` segment in the path is for) — not because Kubernetes archives history generally.

## Collection Architecture Patterns

Since the files above are node-local and rotated/GC'd on a schedule you don't control, something has to move that data off the node before it disappears. The standard shape is a **node agent running as a DaemonSet** — one collector pod per node, reading the same files `kubectl logs` reads, discovering which pod/container each file belongs to via the Kubernetes API, and shipping the parsed lines to a central store. This needs no cooperation from the application at all — it just reads what the runtime was already writing.

## Worked Example: Fluent Bit → Loki → Grafana

1. **Fluent Bit** runs as a DaemonSet, one pod per node.

2. It tails `/var/log/containers/*.log` — the same files `kubectl logs` proxies to. The filename alone already hands it four structural facts, for free, with zero API calls:

   ```
   <pod-name>_<namespace>_<container-name>-<container-id>.log
   ```

   Pod name, namespace, container name, and the container runtime's own ID for that container. Enough to say *which* container a line came from.

3. That's not enough to say *what that pod is*, so Fluent Bit's `kubernetes` filter plugin makes a targeted lookup for each new pod/container it encounters — a `GET` against either the central API server or, more commonly, the **local kubelet's** read-only endpoint on that same node (the kubelet already keeps the full spec of every pod scheduled to it in memory, so asking it locally is cheaper and lower-latency than going to the central API server from every node's DaemonSet pod). The result is cached in-process, keyed by pod, with a TTL — so it's a one-time-per-pod lookup-and-cache, not a request per log line, and not a persistent watch stream either.

   **Why this lookup is necessary at all — the filename genuinely can't carry the answer:**

   - **Labels are arbitrary and unknowable in advance.** `app=backend`, `tier=api`, whatever labels you chose — the kubelet has no way to predict, when it first creates the log file at container start, which key-value pairs you'll later want to filter or group by in Grafana. A fixed filename format can only encode fields that are structural and universal (namespace/pod/container/container-id are always present, on every pod, in the same shape); it can't encode a set of fields whose names and values are entirely user-defined per workload.
   - **Labels can change after the file already exists.** `kubectl label pod ...` mutates a running pod's labels at any time. The log file's name is fixed once, at creation — even if the kubelet *had* tried to bake labels into it, that snapshot would go stale the moment someone re-labeled the pod. Only a live lookup reflects current state.
   - **Owner references, annotations, and pod IP have the same problem** — none of it is structural/fixed the way namespace+pod+container are, so none of it can live in a naming convention either.

   This is the concrete reason the pipeline needs a Kubernetes API interaction at all, rather than being pure file-tailing: the file path is a *key* (enough to identify the source), not the *record* (the metadata you actually want attached for querying).

4. Parsed lines get labeled with both the filename-derived fields and the looked-up metadata (labels, in particular — this is what lets you later write a Loki query like `{app="backend"}` rather than only `{pod="backend-7d9f8b6c9-x2kvq"}`), then pushed to **Loki**, which indexes the labels and stores the compressed log content as chunks.

5. **Grafana** queries Loki and can correlate logs with metrics for the same pod, because both were tagged with the same Kubernetes labels.

## Is There a Risk of Losing Logs?

Yes, in the basic cases:

| Case | What happens | Logs lost? |
|---|---|---|
| **Container restarts** (crash, OOM), same Pod | kubelet keeps the previous instance's log file | No — recoverable via `kubectl logs --previous`, one restart back |
| **Pod moved or replaced** (rescheduled, rolling update, node drained) | kubelet garbage-collects the old Pod's entire log directory once the Pod object is gone | Yes — nothing left to read at all, `--previous` included |

The general rule: node-local log files are always temporary — by rotation if nothing else happens to the pod first, and certainly once the pod itself is gone. So the only way to actually keep application logs is to ship them off the node to somewhere durable (Loki, above, or any equivalent) before that happens. Without a shipper, "how long do my logs survive" is really "how long does this exact pod, on this exact node, keep existing" — which in a system designed to reschedule pods routinely is not very long.

## Key Interactions

- **DaemonSet placement isn't automatic on tainted nodes.** A DaemonSet pod doesn't tolerate arbitrary custom taints by default. Taint nodes for workload isolation without updating the collector's tolerations, and those nodes' pods go unshipped — silently.
- **Shared labels are what make correlation possible.** Grafana pivoting from a metrics panel to matching logs works only because both pipelines tag their data with the same namespace/pod identity independently.
