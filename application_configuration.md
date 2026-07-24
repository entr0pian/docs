# Application Configuration: Delivery Mechanisms and Mutability

Covers how environment variables and ConfigMap/Secret data actually reach a running container, why most of a Pod's spec resists live updates, and the patterns real teams use to make a config change take effect without a human manually deleting Pods.

---

## Can Config Be Applied Without Recreating the Pod?

> **Question:** Can application configurations such as environment variables or ConfigMap updates be applied dynamically without recreating the Pod?

**Short answer: it depends entirely on *how* the config is consumed. Env vars and volume mounts resolve a ConfigMap reference through two genuinely different mechanisms, and neither one gives you what most people assume "dynamic config" means.**

### Two Consumption Modes, Two Different Mechanics

| Mode | What's stored in the Pod spec | When it's resolved | Updates live when the ConfigMap changes? |
|---|---|---|---|
| Env var (`valueFrom.configMapKeyRef` / `envFrom`) | A reference only — ConfigMap name (+ key) | Once, by the kubelet, at container start | **Never.** Frozen until the Pod is recreated. |
| Volume mount | A reference only — ConfigMap name | Kubelet keeps a live watch on it | **Yes**, the file on disk updates in place — but the app must notice on its own. |

Both modes store nothing but a pointer in the Pod spec. `kubectl get pod -o yaml` will never show you the actual ConfigMap contents in either case — only the reference. What differs is what the kubelet does with that reference *after* the Pod starts running.

### Env Var Case: A Reference Resolved Once, Never Revisited

```yaml
env:
  - name: DATABASE_HOST
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: db_host
```

The kubelet fetches the ConfigMap and injects the literal value into the container's process environment (via the container runtime) at the moment it starts the container. That's a one-time, one-directional read — there is no watch, no callback, nothing tying the running process back to the ConfigMap afterward. Edit the ConfigMap a thousand times; the container's env var doesn't move.

```
Container starts
  → kubelet resolves configMapKeyRef → injects literal value into process env
  → connection to the ConfigMap ends here, permanently, for this container's lifetime
ConfigMap edited later
  → Pod spec: unchanged (still just the reference)
  → running process env: unchanged (still the old value)
  → nothing observes the drift; nothing is "wrong" from Kubernetes' point of view
```

Failure mode worth knowing: if the referenced ConfigMap or key doesn't exist at container-start time, the container never starts — you'll see `CreateContainerConfigError` — unless the reference is marked `optional: true`.

### Volume Mount Case: Kubernetes Updates the File — That's Where It Stops

```yaml
volumes:
  - name: config-vol
    configMap:
      name: app-config
containers:
  - volumeMounts:
      - name: config-vol
        mountPath: /etc/app-config
```

Here the kubelet keeps a live source for the ConfigMap (watch-based by default, with a TTL-cache fallback mode) and rewrites the mounted files when it changes — typically within about a minute, no container restart, no Pod recreation. The rewrite itself is atomic: new content lands in a fresh timestamped directory, then a `..data` symlink is atomically repointed at it, and the visible files are symlinks through `..data/<key>`. The app can never observe a half-written file — only a clean old version or a clean new one.

Two gotchas that are easy to walk into:

- **`subPath` silently breaks this.** Mounting a single key via `subPath` (common when you want one config file to sit next to other files in an existing directory) bind-mounts that file directly instead of routing it through the symlink-swap mechanism. It freezes at container-start value — no error, no warning, just permanent staleness that looks identical to the working case until you go looking.
- **`immutable: true` ConfigMaps opt out entirely.** The API server refuses further edits to an immutable ConfigMap's data, and the kubelet stops watching it — a deliberate scalability trade-off. If you need to change it, you create a new ConfigMap under a new name and roll the Deployment, which lands you right back in "cattle" territory on purpose.

Even in the success case, updating the file is as far as Kubernetes goes. Nothing tells the process the file changed — if the app read its config once into memory at startup, it's exactly as stale as the env var case. Actual hot-reload requires the app (or something alongside it) to watch the file — inotify, polling, or a reload signal.

---

## Why Recreation Is the Default: Pod Spec Immutability

Once a Pod is admitted, the API server rejects most changes to its `spec` outright — this is enforced by validation, not just convention, which is why the Deployment controller doesn't even attempt an in-place patch when the Pod template changes. It always creates fresh Pod objects, because that's the only path that's guaranteed to work for *any* field that changed.

A short list of the fields that genuinely can be mutated on a live Pod without recreating it:

| Field | Effect |
|---|---|
| `metadata.labels` / `metadata.annotations` | Always mutable — metadata, not spec. No restart of anything. |
| `spec.containers[*].image` | Mutable — kubelet stops and starts *just that container*. Same Pod UID, same IP, same Node. Container recreation, not Pod recreation. |
| `spec.tolerations` | Additions only, not removal or modification. |
| `spec.activeDeadlineSeconds` | Mutable. |
| `spec.containers[*].resources` (CPU/memory) | Mutable via in-place Pod vertical scaling (`InPlacePodVerticalScaling` feature gate, beta in recent Kubernetes versions). Depending on the container's `resizePolicy`, this can apply without even a container restart. |

Everything else — env vars, volumes, ports, most of the container spec — is immutable for the lifetime of the Pod object. There is no code path in the API server for rewriting a running container's environment or mounts; the only way to change them is a new Pod.

---

## Patterns Mature Teams Use for Config Hot-Reload

Since neither consumption mode gives you an automatic "detect the change and apply it" pipeline, teams that need configuration changes to reliably take effect reach for one of three patterns.

### 1. Checksum/hash annotation — force a rollout on ConfigMap change

The standard Helm pattern:

```yaml
template:
  metadata:
    annotations:
      checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

The rendered ConfigMap's content is hashed and embedded as a Pod template annotation. Annotations are part of `PodTemplateSpec`, so when the ConfigMap's content changes, the hash changes, the Deployment controller sees a template diff — even though the env var/volume reference itself never changed — and runs a normal rolling update. This isn't hot-reload; it's automating the manual recreation you'd otherwise do by hand, triggered by content rather than by a version bump.

### 2. External reload controllers (e.g. Stakater Reloader)

Instead of baking the checksum in at render time, a controller watches ConfigMaps/Secrets at runtime and patches a rollout-triggering annotation onto the Deployments that reference them when it detects a change. Same underlying mechanism as the checksum pattern — force a `PodTemplateSpec` diff — just triggered continuously instead of only at deploy time. This matters when a ConfigMap can be edited directly (`kubectl edit`) outside the normal CD pipeline, where a baked-in checksum wouldn't be recomputed.

### 3. Sidecar/watcher — the only genuinely hot path

When Pod/container recreation truly isn't acceptable, mount the config as a volume (to get the live-updated file) and run a small watcher — as a sidecar or in-process — that inotify-watches the file and triggers the main process to reload. The widely used `configmap-reload` sidecar for nginx is the canonical example: it detects the mounted file changing and sends `nginx -s reload`. This is the only approach of the three that recreates nothing at all — and it only works if the app (or its sidecar) actually has a live-reload primitive to call.

### Summary

| Approach | Pod recreated? | Container restarted? | Requires app support? | Typical use |
|---|---|---|---|---|
| Nothing (raw env var) | — | — | — | unintentional staleness |
| Checksum annotation | Yes | Yes (new Pods) | No | default safe pattern (Helm/Kustomize) |
| Reloader-style controller | Yes | Yes (new Pods) | No | edits made outside the CD pipeline |
| Sidecar/inotify watcher | No | No | Yes (reload hook) | nginx, HAProxy, apps with native reload support |

---

## Takeaway

Nothing in Kubernetes makes configuration "dynamic" by default. Env vars are frozen at container start; the one path that is genuinely live — volume-mounted files — only updates the bytes on disk, not the app's in-memory state. Every real "dynamic config" setup in production is one of two things: automating the recreation so it happens on content-change instead of by hand (checksum annotation, Reloader), or building an actual reload path into the app so recreation is never needed (sidecar/inotify). Both are working around the same underlying fact — nearly everything about how a Pod is configured is immutable by API-server validation the moment the Pod exists.
