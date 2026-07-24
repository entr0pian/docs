# Crossplane Architecture

## What is Crossplane

Crossplane is a Kubernetes operator that lets you provision and manage external cloud infrastructure (AWS, GCP, Azure, etc.) using Kubernetes CRs. Instead of running Terraform or clicking in the AWS console, you apply a YAML manifest and Crossplane reconciles the real resource in the cloud.

---

## Layer 1 — Crossplane Core (the operator)

The first thing installed is the Crossplane core operator. This is a standard Helm chart deployed to `crossplane-system`.

What it does:
- Runs the main Crossplane controllers in the cluster
- Installs the base CRDs: `Provider`, `Configuration`, `Function`, `ProviderConfig`, etc. (the *meta* types — not AWS-specific ones)
- Watches for `Provider` CRs and manages their lifecycle (pulling, installing, health-checking)

What it does NOT do:
- It has no knowledge of AWS, SQS, S3, or any cloud resource
- You cannot create a `Queue` or any cloud resource yet — the CRDs for those don't exist

**Crossplane core is a prerequisite for everything else.**

---

## Layer 2 — The Provider (e.g. provider-aws-sqs)

A Provider is a Crossplane plugin packaged as an OCI image. You install it by applying a `Provider` CR:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-sqs
spec:
  package: xpkg.upbound.io/upbound/provider-aws-sqs:v1.14.0
```

### The `package` field

`package` is an OCI image reference — the same format as a Docker image:

```
xpkg.upbound.io/upbound/provider-aws-sqs:v1.14.0
│                │        │                │
registry         org      image name       tag
```

The image bundles everything needed to run the provider: the Go controller binary, all CRD schemas, and RBAC rules. Crossplane manages the pod lifecycle from it — you don't write a `Deployment` yourself.

When Crossplane core sees this CR it:
1. Pulls the OCI package from the registry
2. Unpacks it and installs the provider-specific CRDs into the cluster (e.g. `Queue`, `QueuePolicy`)
3. Starts the provider controller pod (which watches for `Queue` CRs and calls the AWS SQS API)
4. Sets `Installed=True` and `Healthy=True` on the Provider status once ready

**The provider takes 30–60 seconds to become Healthy** — the CRDs do not exist until that process completes.

### Provider dependency: provider-family-aws

`provider-aws-sqs` declares a dependency inside its OCI image metadata:

```yaml
dependsOn:
  - provider: xpkg.upbound.io/upbound/provider-family-aws
    version: ">=v2.0.0"
```

Crossplane's package manager reads this metadata after pulling the image and automatically creates a second `Provider` object for `provider-family-aws`. You never write that manifest — it appears in `kubectl get providers` as a side effect.

**Why this split exists:**

Before this architecture, Upbound shipped one giant `provider-aws` that contained every AWS service — S3, EC2, RDS, IAM, SQS, and hundreds more. That image registered ~900 CRDs into the cluster even if you only needed one service. It was enormous and slow to install.

Upbound solved this by splitting AWS into two layers:

| Provider | Responsibility |
|---|---|
| `provider-family-aws` | Shared auth machinery, the `ProviderConfig` CRD, IAM role assumption, region config — everything common across all AWS services |
| `provider-aws-sqs` | Only the SQS-specific CRDs (`Queue`, `QueuePolicy`) and the SQS controller |

When your SQS controller needs to make an AWS API call, it delegates all credential handling up to the family provider. The SQS provider itself has no auth logic — it only knows SQS.

The result: installing `provider-aws-sqs` registers ~5 CRDs instead of ~900, and the family provider is small since it only brings the shared auth CRDs.

---

## Layer 3 — The ProviderConfig

Once the Provider is Healthy (and its CRDs exist), you can apply a `ProviderConfig`:

```yaml
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: crossplane-aws-credentials
      key: creds
```

What it does:
- Tells the provider **how to authenticate** to AWS
- Points to a Kubernetes Secret containing the AWS credentials in CLI credentials file format (`[default]\naws_access_key_id=...\naws_secret_access_key=...`)
- Named `default` — any managed resource (e.g. `Queue`) that does not explicitly reference a ProviderConfig will use this one

**The ProviderConfig CRD (`aws.upbound.io/v1beta1`) only exists after the Provider is Healthy.** Trying to apply it before that will fail with "resource not found in cluster".

---

## Layer 4 — The credentials Secret

The ProviderConfig references a Secret that must exist before the ProviderConfig (or any managed resource) is used. In this project it is synced from AWS Secrets Manager by ESO:

```
AWS SM: taskapp/{env}/crossplane-aws
  → ExternalSecret (in platform chart)
    → Secret: crossplane-aws-credentials (in crossplane-system)
      → ProviderConfig references it
```

The IAM user behind these credentials has a least-privilege policy scoped only to SQS operations (`CreateQueue`, `DeleteQueue`, `GetQueueUrl`, etc.).

---

## Full dependency chain

```
1. crossplane core (Helm chart)
       ↓ installs Provider CRD and controllers
2. Provider CR (provider-aws-sqs)
       ↓ Crossplane pulls OCI package
       ↓ installs SQS CRDs (Queue, etc.)
       ↓ starts provider controller pod
       ↓ becomes Healthy
3. ProviderConfig CR (default)
       ↓ references crossplane-aws-credentials Secret
       ↓ provider uses it to auth to AWS
4. Queue CR (managed resource)
       ↓ provider controller reconciles it
       ↓ SQS queue created in AWS
```

---

## Install ordering in this project

| Wave | What | Why |
|---|---|---|
| 0 | `crossplane` (core) | Must exist before Provider CR can be applied |
| 0 | `external-secrets` | Must exist before ExternalSecret can sync the credentials Secret |
| 1 | `taskapp-platform` | Applies Provider CR + ExternalSecret for credentials |
| 2 | `crossplane-provider-config` | Applies ProviderConfig — safe only after Provider is Healthy and credentials Secret exists |

Wave 2 is the key gate: by the time `crossplane-provider-config` syncs, the Provider CRD exists and the Secret is populated.

---

## Why the ProviderConfig is a separate ArgoCD app

The `ProviderConfig` CRD is installed by the Provider package — not by Crossplane core and not by a CRD manifest in any of our charts. ArgoCD validates all resources in an Application before applying any of them. If the ProviderConfig is in the same Application as the Provider, ArgoCD fails validation at sync time because the CRD does not exist yet.

Putting the ProviderConfig in a separate wave-2 Application guarantees ArgoCD only attempts to apply it after wave 1 (which includes the Provider) is fully Healthy.

---

## Compositions and XRDs — The Infrastructure API Layer

Crossplane's managed resources (like `Queue`, `SecurityGroup`, `Instance`) are low-level primitives — one object per AWS resource. For real workloads you want a higher-level abstraction: "give me a database" rather than "give me a SecurityGroup, then a SubnetGroup, then an RDS Instance." That is what XRDs and Compositions provide.

### The three objects

| Object | Kind | Who writes it | Purpose |
|---|---|---|---|
| `CompositeResourceDefinition` (XRD) | `apiextensions.crossplane.io/v1` | Platform engineer | Defines the API schema — what parameters exist, their types, and defaults |
| `Composition` | `apiextensions.crossplane.io/v1` | Platform engineer | Defines the blueprint — which managed resources to create and how to map XR fields to them |
| `XRDSInstance` / `RDSInstance` | `database.taskapp.io/v1alpha1` | App team or automation | The actual request — "give me one of these with these parameters" |

---

### Step-by-step: from XRD to running RDS instance

#### 1. Apply the XRD

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xrdsinstances.database.taskapp.io
spec:
  group: database.taskapp.io
  names:
    kind: XRDSInstance        # cluster-scoped composite
  claimNames:
    kind: RDSInstance         # namespace-scoped claim
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema: ...             # openAPIV3Schema with parameters
```

**What Crossplane does immediately:**

The `apiextensions` controller reads the XRD and registers **two real CRDs** with the Kubernetes API server:

```
xrdsinstances.database.taskapp.io   (cluster-scoped)  → XRDSInstance
rdsinstances.database.taskapp.io    (namespace-scoped) → RDSInstance (the Claim)
```

These are now queryable exactly like native Kubernetes resources:

```bash
kubectl api-resources | grep taskapp
# xrdsinstances   database.taskapp.io/v1alpha1   false   XRDSInstance
# rdsinstances    database.taskapp.io/v1alpha1   true    RDSInstance

kubectl explain xrdsinstance.spec.parameters
# Shows dbName (required), region, vpcId, subnetIds, instanceClass, storageGB
```

No XR has been created yet — only the schema exists.

---

#### 2. Apply the Composition

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xrdsinstances.database.taskapp.io
spec:
  compositeTypeRef:
    apiVersion: database.taskapp.io/v1alpha1
    kind: XRDSInstance          # binds this Composition to the XRD above
  resources:
    - name: security-group
      base:
        kind: SecurityGroup
        ...
    - name: security-group-rule
      ...
    - name: subnet-group
      ...
    - name: instance
      ...
```

**What Crossplane does immediately:**

1. Validates the `compositeTypeRef` — does `XRDSInstance` exist as an XRD? Yes.
2. Validates each composed resource kind — do `SecurityGroup`, `SecurityGroupRule`, `SubnetGroup`, `Instance` CRDs exist (i.e. are the providers installed)? Yes.
3. Creates an immutable **CompositionRevision** snapshot. Every update to the Composition creates a new revision. XRs can be pinned to a specific revision or set to `Automatic` (always track the latest).

```bash
kubectl get compositionrevision
# xrdsinstances.database.taskapp.io-77eb787   revision 1   (original)
# xrdsinstances.database.taskapp.io-13dc250   revision 2   (after a fix)
```

The Composition is now standing by — watching for any `XRDSInstance` object to be created.

---

#### 3. Apply the XR

```yaml
apiVersion: database.taskapp.io/v1alpha1
kind: XRDSInstance
metadata:
  name: taskapp-prod-rds
spec:
  parameters:
    dbName: taskapp          # only required field; everything else uses XRD defaults
```

**What Crossplane does:**

The composite reconciler picks this up and runs its loop:

```
XRDSInstance/taskapp-prod-rds created
  │
  ├─ Find matching Composition via compositeTypeRef
  ├─ Stamp compositionRevisionRef onto the XR (locks it to current revision)
  │
  ├─ Apply XRD defaults to missing parameters:
  │    region:        eu-west-1
  │    vpcId:         vpc-0274e6e3e08a56006
  │    subnetIds:     [subnet-0782..., subnet-07a2..., subnet-0a55...]
  │    instanceClass: db.t3.micro
  │    storageGB:     20
  │
  ├─ Create all 4 composed objects simultaneously:
  │    SecurityGroup/taskapp-prod-rds-{hash}      ← patches: region, vpcId
  │    SecurityGroupRule/taskapp-prod-rds-{hash}  ← patches: region + matchControllerRef for SG
  │    SubnetGroup/taskapp-prod-rds-{hash}        ← patches: region, subnetIds
  │    Instance/taskapp-prod-rds-{hash}           ← patches: region, instanceClass, storageGB, dbName
  │
  └─ Set ownerReference on each composed object:
       ownerReferences:
         - kind: XRDSInstance
           name: taskapp-prod-rds
           controller: true       ← this is what matchControllerRef: true resolves against
```

---

### The reconciliation loop in detail

There are three independent reconcile loops running simultaneously after the XR is applied:

```
┌─────────────────────────────────────────────────────────┐
│  Loop 1 — Crossplane composite reconciler               │
│  Watches: XRDSInstance                                  │
│                                                         │
│  Every reconcile pass:                                  │
│  1. Read XR spec + composition blueprint                │
│  2. For each composed resource:                         │
│     - Does it exist? No → create it                     │
│     - Yes → re-apply patches (in case XR spec changed)  │
│  3. Collect Ready conditions from all composed objects  │
│  4. Write aggregate Ready status back to the XR         │
└─────────────────────────────────────────────────────────┘
         │ creates/updates composed objects
         ▼
┌──────────────────────────────┐  ┌──────────────────────────────┐
│  Loop 2a — provider-aws-ec2  │  │  Loop 2b — provider-aws-rds  │
│  Watches: SecurityGroup,     │  │  Watches: SubnetGroup,        │
│           SecurityGroupRule  │  │           Instance            │
│                              │  │                               │
│  For each object:            │  │  For each object:             │
│  1. Observe (AWS API)        │  │  1. Observe (AWS API)         │
│  2. Compare vs spec          │  │  2. Compare vs spec           │
│  3. Create/Update in AWS     │  │  3. Create/Update in AWS      │
│  4. Write status.atProvider  │  │  4. Write status.atProvider   │
│  5. Set Ready condition      │  │  5. Set Ready condition        │
└──────────────────────────────┘  └──────────────────────────────┘
```

Loop 1 does not wait for loop 2 to finish. It creates all 4 composed objects in one pass, then re-reconciles periodically. As each AWS resource becomes ready, loop 2 sets `Ready: True` on the composed object, and loop 1 picks that up on its next pass and updates the XR's aggregate status.

**Cross-resource dependency resolution** (e.g. SecurityGroupRule needs the SG's ID):

The `securityGroupIdSelector: matchControllerRef: true` field on the SecurityGroupRule tells the EC2 provider: "find a SecurityGroup whose `ownerReference.controller` is the same XR that owns me." The provider keeps retrying until the SecurityGroup exists and has a populated `status.atProvider.id`. No explicit ordering is needed — the retry loop handles it naturally.

**Status propagation:**

```
AWS resource becomes available
  → provider sets Ready: True on composed object
    → composite reconciler sees all composed objects Ready
      → sets Ready: True on XRDSInstance
```

---

### XR vs Claim — when to use which

**XR (`XRDSInstance`)** is cluster-scoped. Platform engineers or GitOps (ArgoCD) apply it directly when provisioning infrastructure at the cluster level.

**Claim (`RDSInstance`)** is namespace-scoped. Application teams apply it inside their namespace without needing cluster-level RBAC. Crossplane creates the corresponding XR automatically behind the scenes. This is the multi-tenancy pattern — devs get a simple namespaced API and never touch the cluster-scoped XR.

---

## Drift Detection and Correction

### How it works

Every managed resource (e.g. `Queue`) has a dedicated controller that runs a continuous reconciliation loop. The loop has three phases:

**1. Observe** — the controller calls the AWS API to read the actual current state of the resource:
- `sqs:GetQueueAttributes` — reads queue configuration (visibility timeout, retention period, etc.)
- `sqs:ListQueueTags` — reads the current tags

**2. Compare** — the controller diffs the observed AWS state against `spec.forProvider` in the CR. If they match, nothing happens and the loop sleeps until the next poll.

**3. Correct** — if drift is found, the controller immediately calls the AWS API to bring reality back in line (`sqs:SetQueueAttributes`, `sqs:TagQueue`, etc.). This happens in the same reconcile iteration — no queuing, no delay.

After the correction is confirmed by AWS, the controller writes back to the Kubernetes API:
- `status.atProvider` — reflects the actual state of the resource in AWS (queue URL, ARN, all attributes)
- `status.conditions` — `Synced=True/False`, `Ready=True/False`

### The correction flow

```
Kubernetes CR (spec.forProvider = desired state)
      ↓ controller reads desired state
AWS API — Observe (GetQueueAttributes, ListQueueTags)
      ↓ drift detected
AWS API — Correct (SetQueueAttributes, TagQueue)
      ↓ AWS confirms update
Kubernetes CR status updated (status.atProvider, conditions)
```

Kubernetes never talks to AWS directly. The controller mediates both directions — it reads desired state from Kubernetes and actual state from AWS, then reconciles the two.

### Poll interval

By default Crossplane polls every **10 minutes** (`--poll-interval` flag on the provider controller). So if someone modifies a queue attribute directly in the AWS console, Crossplane will detect and revert it within 10 minutes.

This is the fundamental difference from Terraform: Terraform only detects drift when you explicitly run `terraform plan`. Crossplane is always watching and always correcting.

### Opting out of correction

You can control how aggressively Crossplane manages a resource via `spec.managementPolicies` on the CR:

| Policy | Behaviour |
|---|---|
| `["*"]` (default) | Full control — create, observe, update, delete |
| `["Observe"]` | Watch only — never correct drift |
| `["Observe", "Create"]` | Creates the resource but never updates it after that |

### Cluster deletion caveat

Crossplane uses Kubernetes finalizers to ensure AWS resources are deleted before the CR is removed. However, if the cluster itself is hard-deleted (e.g. `kind delete cluster`), the controller process is killed immediately — finalizers are bypassed and the AWS resource is left orphaned. Always delete managed resource CRs and wait for Crossplane to confirm deletion before tearing down the cluster.

---

## Composition Field Reference

This section covers the widely used fields inside a `Composition` beyond the basic `base` + `patches` pattern.

---

### Patch types

A `Composition` resource entry can have multiple patches. The direction and shape of each patch is controlled by `type`.

#### `FromCompositeFieldPath` (most common)

Pulls a value from the XR/claim and writes it into the composed resource.

```yaml
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.parameters.region
    toFieldPath: spec.forProvider.region
```

#### `ToCompositeFieldPath`

The reverse: reads a value from the composed resource (typically from `status.atProvider`) and writes it back to the XR's `status`. This is how you surface outputs — like an RDS endpoint — back to the claim so the application can consume them.

```yaml
patches:
  - type: ToCompositeFieldPath
    fromFieldPath: status.atProvider.endpoint
    toFieldPath: status.atProvider.endpoint
```

After this patch the XR will have `status.atProvider.endpoint` set to the real AWS endpoint. The claim mirrors the XR status automatically.

#### `CombineFromComposite`

Combines multiple XR fields into a single string and writes the result to the composed resource. Useful for naming conventions.

```yaml
patches:
  - type: CombineFromComposite
    combine:
      variables:
        - fromFieldPath: metadata.name
        - fromFieldPath: spec.parameters.env
      strategy: string
      string:
        fmt: "%s-%s-db"     # e.g. taskapp-prod-db
    toFieldPath: spec.forProvider.identifier
```

#### `PatchSet` reference

References a named patchSet defined at the top of the Composition (see patchSets section below). Used to avoid repeating the same patches on every resource.

```yaml
patches:
  - type: PatchSet
    patchSetName: common
```

---

### `transforms` on patches

Any `FromCompositeFieldPath` or `ToCompositeFieldPath` patch can include a `transforms` list that modifies the value before writing it.

#### `convert` — type coercion

```yaml
patches:
  - type: FromCompositeFieldPath
    fromFieldPath: spec.parameters.storageGB
    toFieldPath: spec.forProvider.allocatedStorage
    transforms:
      - type: convert
        convert:
          toType: int64
```

#### `map` — enum translation

Maps an abstract value (e.g. `small`) to a provider-specific string (e.g. `db.t3.micro`). Keeps the claim API friendly while the provider gets what it expects.

```yaml
transforms:
  - type: map
    map:
      small: db.t3.micro
      medium: db.t3.small
      large: db.t3.medium
```

#### `string` — format with a template

```yaml
transforms:
  - type: string
    string:
      fmt: "taskapp-%s"     # prepends a prefix
```

#### `math` — arithmetic

```yaml
transforms:
  - type: math
    math:
      multiply: 1024        # convert GiB to MiB for example
```

---

### `patchSets` — DRY shared patches

Fields like `region` and `metadata.name` are typically patched on every resource in the Composition. Rather than repeating them, define a named patchSet once and reference it per resource.

```yaml
spec:
  patchSets:
    - name: common
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: metadata.name
          toFieldPath: metadata.name
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.region

  resources:
    - name: security-group
      patches:
        - type: PatchSet
          patchSetName: common
        - type: FromCompositeFieldPath      # resource-specific patches follow
          fromFieldPath: spec.parameters.vpcId
          toFieldPath: spec.forProvider.vpcId
```

---

### `connectionDetails` — surfacing credentials as a Secret

After Crossplane provisions a resource (e.g. an RDS instance), you typically want the connection details (host, port, username, password) available as a Kubernetes Secret so application pods can mount them. Each resource in the Composition can declare which fields to collect:

```yaml
resources:
  - name: instance
    connectionDetails:
      - type: FromFieldPath
        name: endpoint
        fromFieldPath: status.atProvider.endpoint
      - type: FromFieldPath
        name: port
        fromFieldPath: status.atProvider.port
      - type: FromValue
        name: username
        value: taskuser
```

`type: FromFieldPath` reads from the composed resource's status (populated by the provider after the resource exists in AWS). `type: FromValue` writes a static value.

Crossplane merges connection details from all resources in the Composition into a single Secret. To enable writing, two things must be set:

**On the Composition:**
```yaml
spec:
  writeConnectionSecretsToNamespace: crossplane-system
```

**On the XR or claim:**
```yaml
spec:
  writeConnectionSecretToRef:
    namespace: default
    name: rds-connection
```

The resulting Secret (named `rds-connection` in `default`) will contain keys `endpoint`, `port`, `username` — ready to be mounted as env vars or a volume by application pods.

---

### `readinessChecks` — when is a composed resource considered ready?

By default Crossplane marks a composed resource ready when its `status.conditions` contains `Ready=True`. For long-running provisions (RDS instance creation takes several minutes) you can override this to check a more specific field:

```yaml
resources:
  - name: instance
    readinessChecks:
      - type: MatchString
        fieldPath: status.atProvider.dbInstanceStatus
        matchString: available
```

The XR stays `Ready: False` until every composed resource passes its readiness check. This prevents the XR from appearing healthy before the RDS instance is actually accepting connections.

Available check types:

| Type | Description |
|---|---|
| `MatchString` | Field must equal an exact string |
| `MatchTrue` | Field must be `"true"` |
| `MatchFalse` | Field must be `"false"` |
| `NonEmpty` | Field must exist and be non-empty |
| `None` | Always ready — skip readiness checking for this resource |

---

### `compositionRevisionRef` and update policies

Every time you update a Composition, Crossplane creates an immutable `CompositionRevision` snapshot. Existing XRs can be configured to track updates automatically or stay pinned to the revision they were created with.

```yaml
spec:
  compositionUpdatePolicy: Automatic   # default — XR always uses the latest revision
  # compositionUpdatePolicy: Manual    # XR stays on current revision until explicitly updated
```

With `Manual`, rolling out a composition change is a controlled operation — you update each XR's `compositionRevisionRef` explicitly, making canary-style rollouts possible.

```bash
kubectl get compositionrevision
# xrdsinstances.database.taskapp.io-77eb787   revision 1
# xrdsinstances.database.taskapp.io-13dc250   revision 2   ← latest
```

---

### `publishConnectionDetailsWithStoreConfigRef` — external secret stores

Instead of writing connection details to a plain Kubernetes Secret, Crossplane can push them directly to an external store (Vault, AWS Secrets Manager) via the External Secret Store (ESS) plugin:

```yaml
spec:
  publishConnectionDetailsWithStoreConfigRef:
    name: vault-store     # references a StoreConfig CR
```

This is an advanced pattern — it requires the ESS plugin installed alongside Crossplane and a `StoreConfig` CR that defines the target store and auth. The default (no field set) writes to a Kubernetes Secret.

---

### Complete field map

```
Composition.spec
├── compositeTypeRef          — links this Composition to its XRD
├── patchSets[]               — reusable named groups of patches
├── writeConnectionSecretsToNamespace  — where to write the merged connection Secret
├── publishConnectionDetailsWithStoreConfigRef  — push to Vault/SM instead
└── resources[]
    ├── name                  — logical name for this composed resource
    ├── base                  — the full managed resource spec (static defaults)
    │   └── spec.forProvider  — the AWS resource configuration
    ├── patches[]             — dynamic values sourced from the XR
    │   ├── type              — FromCompositeFieldPath | ToCompositeFieldPath | CombineFromComposite | PatchSet
    │   ├── fromFieldPath     — source field path
    │   ├── toFieldPath       — destination field path
    │   └── transforms[]      — optional value transformations (convert, map, string, math)
    ├── connectionDetails[]   — which fields to collect into the connection Secret
    └── readinessChecks[]     — when this resource is considered ready
```

---

## Composition Functions and Pipeline Mode

### The problem with built-in P&T

The `resources` + `patches` model described above is called **Patch & Transform (P&T) mode**. It is a fixed DSL interpreted directly by the Crossplane composite reconciler. The logic it supports is what Crossplane ships — you cannot add conditionals, loops, or dynamic derivations that P&T does not natively provide.

Two concrete limitations:

1. **XR self-patching is unreliable.** `ToCompositeFieldPath` can write back to the XR, but in P&T mode Crossplane processes connection secret writes before it applies the patched XR state. This means you cannot reliably use a managed resource's field to derive something like `spec.writeConnectionSecretToRef.name` on the XR itself.

2. **Logic is not composable.** If you need a second transformation step that builds on the output of the first — for example, first resolving a VPC ID from a tag, then using that ID to place a subnet — P&T has no way to chain operations.

### How pipeline mode works

In pipeline mode, the built-in P&T engine is replaced by a gRPC-based plugin system. Instead of Crossplane interpreting `resources` inline, you declare a `pipeline` of steps. Each step names a **Function** — a separate pod that Crossplane calls over gRPC during every reconcile pass.

```
XR reconcile triggered
  │
  ├─ Crossplane sends current observed state to Function 1
  │      Function 1 computes desired state → returns it
  │
  ├─ Crossplane sends that result to Function 2
  │      Function 2 further modifies desired state → returns it
  │
  └─ Crossplane applies the final desired state:
       creates/updates managed resources in the cluster
       writes the XR's connection secret
       updates XR status
```

Each Function is a container image packaged as a `Function` CR — installed and managed by Crossplane exactly like a Provider:

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-patch-and-transform:v0.8.1
```

Crossplane pulls the image, runs it as a pod in `crossplane-system`, and keeps a persistent gRPC connection to it. The Function pod stays running and receives calls on every reconcile.

### `function-patch-and-transform`

This is the official reimplementation of the P&T engine as a Function. Migrating from P&T mode to pipeline mode with only this one function is **functionally equivalent** — same patch types, same transform syntax. The difference is that the logic now runs in the Function pod rather than inside the Crossplane controller, and crucially, **the Function returns the fully resolved XR state in a single shot** before Crossplane acts on it. This makes XR self-patching reliable.

### Transition: P&T mode → pipeline mode

Below is the before and after for the `xrdsinstances` Composition.

**Before (P&T mode):**

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xrdsinstances.database.taskapp.io
spec:
  compositeTypeRef:
    apiVersion: database.taskapp.io/v1alpha1
    kind: XRDSInstance
  writeConnectionSecretsToNamespace: default  # top-level, P&T-only field
  patchSets:
    - name: common
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: metadata.name
          toFieldPath: metadata.name
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.region
  resources:
    - name: instance
      base:
        apiVersion: rds.aws.upbound.io/v1beta1
        kind: Instance
        spec:
          forProvider:
            engine: postgres
            ...
      patches:
        - type: PatchSet
          patchSetName: common
        - type: FromCompositeFieldPath
          fromFieldPath: metadata.name
          toFieldPath: spec.writeConnectionSecretToRef.name
          transforms:
            - type: string
              string:
                fmt: "%s-instance-conn"
      connectionDetails:
        - type: FromFieldPath
          name: endpoint
          fromFieldPath: status.atProvider.address
        ...
```

The user must supply `spec.writeConnectionSecretToRef` on every XR they create — Crossplane needs it to know where to write the final merged connection Secret, and P&T mode cannot derive it automatically.

**After (pipeline mode):**

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xrdsinstances.database.taskapp.io
spec:
  compositeTypeRef:
    apiVersion: database.taskapp.io/v1alpha1
    kind: XRDSInstance
  mode: Pipeline                            # enables pipeline mode
  pipeline:
    - step: patch-and-transform
      functionRef:
        name: function-patch-and-transform  # the Function CR installed above
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        patchSets:                          # same patchSets syntax, now inside input
          - name: common
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: metadata.name
                toFieldPath: metadata.name
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region
        resources:                          # same resources syntax, now inside input
          - name: instance
            base:
              apiVersion: rds.aws.upbound.io/v1beta1
              kind: Instance
              spec:
                forProvider:
                  engine: postgres
                  ...
            patches:
              - type: PatchSet
                patchSetName: common
              - type: FromCompositeFieldPath
                fromFieldPath: metadata.name
                toFieldPath: spec.writeConnectionSecretToRef.name
                transforms:
                  - type: string
                    string:
                      fmt: "%s-instance-conn"
              # NEW: write back to the XR — reliable in pipeline mode
              - type: ToCompositeFieldPath
                fromFieldPath: metadata.name
                toFieldPath: spec.writeConnectionSecretToRef.name
                transforms:
                  - type: string
                    string:
                      fmt: "%s-database-connection-details"
            connectionDetails:
              - type: FromFieldPath
                name: endpoint
                fromFieldPath: status.atProvider.address
              ...
```

The structural change is:
- `writeConnectionSecretsToNamespace` is removed (not used in pipeline mode)
- `patchSets` and `resources` move inside `pipeline[0].input`
- A `ToCompositeFieldPath` patch on the instance derives `spec.writeConnectionSecretToRef.name` on the XR automatically

The XRD also gains a schema default so `namespace` is injected by the Kubernetes API server without the user specifying it:

```yaml
# inside XRD versions[].schema.openAPIV3Schema.properties.spec.properties
writeConnectionSecretToRef:
  type: object
  default:
    namespace: default
  properties:
    name:
      type: string
    namespace:
      type: string
      default: default
```

The result: a user creates an XR with only `metadata.name` and `spec.parameters.dbName`. The connection Secret appears automatically at `<name>-database-connection-details` in `default` — no `writeConnectionSecretToRef` block required.

### Why `ToCompositeFieldPath` works in pipeline mode but not in P&T mode

In P&T mode, Crossplane's composite reconciler runs P&T logic and then immediately proceeds to write the connection Secret — both in the same controller pass. The XR update from `ToCompositeFieldPath` is written to the Kubernetes API asynchronously and is only visible on the *next* reconcile pass. So the connection secret write happens before the patched `writeConnectionSecretToRef.name` is seen.

In pipeline mode, the Function returns the **complete desired XR state** (including any `ToCompositeFieldPath` patches) as a single response object before Crossplane does anything. Crossplane then uses that resolved state — with `writeConnectionSecretToRef.name` already set — when it processes the connection Secret write. Everything happens in one logical step with no async gap.

### Chaining multiple functions

Pipeline mode's real power is chaining. Each step receives the desired state from the previous step and can further modify it:

```yaml
pipeline:
  - step: patch-and-transform
    functionRef:
      name: function-patch-and-transform
    input:
      # standard P&T logic

  - step: auto-ready
    functionRef:
      name: function-auto-ready
    # no input needed — marks XR ready when all composed resources are ready

  - step: custom-logic
    functionRef:
      name: function-go-templating   # community function using Go templates
    input:
      # arbitrary logic: conditionals, loops, dynamic naming
```

Common community Functions:

| Function | Purpose |
|---|---|
| `function-patch-and-transform` | Standard P&T logic in pipeline mode |
| `function-auto-ready` | Marks XR ready when all composed resources are ready |
| `function-go-templating` | Full Go template logic for dynamic composition |
| `function-kcl` | KCL language for complex composition policies |
| `function-sequencer` | Explicit ordering: wait for resource A before creating resource B |