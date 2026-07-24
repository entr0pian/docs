# Pod & Container Lifecycle: Failures and Restarts

Covers what actually happens — container-level vs Pod-level — when a workload fails at runtime: OOM kills, probe failures, crashes, and eviction.

---

## OOM Kills: Container Restart or Pod Recreation?

> **Question:** If an application running in a container encounters an OOM (Out-of-Memory) error, will the container restart, or will the entire Pod be recreated?

**Short answer: the container restarts in place — the Pod is not recreated.**

### What Actually Happens

1. The container exceeds its `resources.limits.memory` (or the Node itself runs out of memory).
2. The kernel's cgroup OOM killer sends `SIGKILL` to the process(es) in that container's cgroup. Only that container dies — sibling containers in the same Pod are untouched.
3. The kubelet observes the container exited, sets its status to `Terminated`, reason `OOMKilled`, exit code `137`.
4. The kubelet restarts **that container** according to the Pod's `restartPolicy` (default `Always`) — same Pod object, same Pod UID, same Node, same IP. `restartCount` increments.
5. Repeated OOM kills produce `CrashLoopBackOff`, with exponential backoff between restart attempts.

```
Container exceeds memory.limit
  → cgroup OOM killer sends SIGKILL
  → kubelet marks container Terminated, reason=OOMKilled, exitCode=137
  → kubelet restarts the container (same Pod, same Node, same IP)
  → restartCount++
  → repeats too fast → CrashLoopBackOff
```

### The Exception: Node-Level Memory Pressure

If the **Node** (not just one container's cgroup) runs low on memory, a different mechanism takes over: kubelet **node-pressure eviction**. This evicts the **entire Pod** — all containers, not just the offending one — to protect Node stability, triggered by the `memory.available` eviction signal.

If that Pod is owned by a Deployment/ReplicaSet/StatefulSet, the controller then creates a **brand-new Pod object**, likely scheduled onto a different Node.

### Comparison

| Trigger | Scope | Mechanism | Result |
|---|---|---|---|
| Container exceeds `limits.memory` | Single container | cgroup OOM killer (`SIGKILL`, exit 137) | Container restarts in place — **same Pod** |
| Node runs low on memory | Whole Pod | kubelet node-pressure eviction | Pod is evicted — controller creates a **new Pod**, possibly on another Node |

### Note on Probes

The cgroup OOM kill bypasses probes entirely — the kernel kills the process directly, regardless of `livenessProbe`/`readinessProbe` configuration. Probes only matter for *hangs without memory growth* (e.g. a deadlock, no crash): only a failing `livenessProbe` would trigger a restart in that case. A Pod with no `livenessProbe` configured — only a `readinessProbe` — will never be restarted automatically for that kind of non-crashing hang; it will simply stop receiving traffic while the process stays alive.

---

## Readiness vs Liveness: Why Only One of Them Can Trigger a Restart

A natural but incorrect assumption: "if `readinessProbe` fails long enough, surely *something* will notice the Pod is broken and replace it — a Deployment controller, a ReplicaSet, something." **Nothing does.** The two probes have strictly separate jobs, and only one of them results in any corrective action.

### Two Different Questions

| Probe | Question it answers | Failure action |
|---|---|---|
| `readinessProbe` | "Should this Pod receive traffic *right now*?" | Pod's IP is removed from the Service's Endpoints/EndpointSlice. **Nothing else happens.** |
| `livenessProbe` | "Should this container be killed and restarted?" | kubelet sends `SIGTERM` (respecting `terminationGracePeriodSeconds`), then `SIGKILL` if it doesn't exit — then restarts the container per `restartPolicy`. Same Pod, `restartCount++`. |

`readinessProbe` is purely a traffic-routing signal. It has no destructive power at all — it cannot kill a container, delete a Pod, or trigger a replacement.

### Why the ReplicaSet Controller Doesn't Step In

This is the piece that makes the misconception non-obvious: a Deployment's `replicas: N` is enforced by a ReplicaSet controller, and that controller decides whether it has "enough" Pods by checking **whether the Pod objects still exist** (i.e. haven't been deleted) — it does **not** check their `Ready` condition. A Pod that is `Running` but perpetually `NotReady` still counts as one of the N replicas. The controller has no reason to create a replacement, because as far as it's concerned, nothing is missing.

### Worked Example: A Deadlock With No `livenessProbe`

Say a container's application logic deadlocks — a goroutine blocks forever on a lock, or a downstream call hangs indefinitely. The process itself never crashes and memory usage never grows, so there is nothing for the kernel's OOM killer to react to (see the section above). Only `readinessProbe` is configured; there is no `livenessProbe`.

```
App deadlocks — process alive, memory flat, no response to requests
  → readinessProbe starts failing
  → after failureThreshold consecutive failures, Ready condition → False
  → Pod's IP removed from Service Endpoints/EndpointSlice
  → traffic stops reaching the Pod
  → ReplicaSet controller: Pod object still exists → requirement satisfied → does nothing
  → Pod sits indefinitely: `1/1 Running`, `READY 0/1`
  → zero traffic, zero automatic recovery, still consuming its requested CPU/memory
```

The Pod becomes silently dead weight. Nothing kills it, nothing replaces it, nothing pages anyone unless there's separate alerting on `kube_pod_status_ready` or similar. It stays exactly like that until a human intervenes (e.g. `kubectl delete pod`) or some unrelated event happens to recycle it (rollout, node replacement, manual restart).

### Same Scenario, With a `livenessProbe` Configured

```
App deadlocks — process alive, memory flat, no response to requests
  → both readinessProbe and livenessProbe start failing
  → readinessProbe fails first (or together) → Pod removed from Service Endpoints
  → livenessProbe crosses its failureThreshold
  → kubelet marks the container unhealthy
  → SIGTERM → grace period → SIGKILL if still running
  → kubelet restarts the container (same Pod, same Node, same IP)
  → restartCount++
  → fresh process comes up, readinessProbe eventually passes → Pod goes back to Ready → traffic resumes
```

### Takeaway

`readinessProbe` protects **traffic** from a broken Pod. `livenessProbe` protects the **Pod** from itself. They are not redundant, and one cannot substitute for the other — a Pod with only a `readinessProbe` has no self-healing path for a hang, deadlock, or any failure mode that doesn't also trip the kernel's OOM killer or cause the process to actually exit.

---
