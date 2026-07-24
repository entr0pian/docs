# KEDA SQS Autoscaling

## What is KEDA?

KEDA (Kubernetes Event-Driven Autoscaling) is a Kubernetes operator that scales workloads based on external event sources — in this case, the number of messages visible in an AWS SQS queue.

KEDA does not replace the HPA. Instead it creates and manages an HPA under the hood, feeding it an external metric value derived from the event source. You never interact with that HPA directly.

---

## Architecture

```
AWS SQS Queue
     │
     │  GetQueueAttributes (ApproximateNumberOfMessages)
     ▼
KEDA Controller  ──── pollingInterval (default 30s) ────► updates external metric
     │
     ▼
  HPA (managed by KEDA)  ──── sync period (15s, cluster-wide) ────► calculates desired replicas
     │
     ▼
taskapp-backend Deployment  ──► pods scaled up or down
```

---

## Credential Flow

The backend already uses an IAM user whose credentials are synced from AWS Secrets Manager into a native Kubernetes Secret via the External Secrets Operator:

```
AWS Secrets Manager (taskapp/prod/backend-credentials)
     │
     │  ESO ClusterSecretStore
     ▼
K8s Secret: taskapp-backend-aws-credentials
  keys: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
     │
     ▼
TriggerAuthentication: taskapp-backend-sqs-auth
     │
     ▼
ScaledObject trigger (authenticationRef)
```

KEDA uses the same IAM credentials the backend pod uses for SQS polling. The IAM user needs `sqs:GetQueueAttributes` on the queue.

---

## Resources

### TriggerAuthentication

Tells KEDA which Kubernetes Secret holds the AWS credentials and how to map its keys to the expected parameters.

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: taskapp-backend-sqs-auth
spec:
  secretTargetRef:
    - parameter: awsAccessKeyID
      name: taskapp-backend-aws-credentials
      key: AWS_ACCESS_KEY_ID
    - parameter: awsSecretAccessKey
      name: taskapp-backend-aws-credentials
      key: AWS_SECRET_ACCESS_KEY
```

### ScaledObject

Defines what to scale, which queue to watch, and the scaling parameters.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: taskapp-backend-scaler
spec:
  scaleTargetRef:
    name: taskapp-backend
  minReplicaCount: 1
  maxReplicaCount: 10
  cooldownPeriod: 300
  triggers:
    - type: aws-sqs-queue
      authenticationRef:
        name: taskapp-backend-sqs-auth
      metadata:
        queueURL: "https://sqs.eu-west-1.amazonaws.com/425832464758/taskapp-prod-tasks"
        queueLength: "5"
        awsRegion: eu-west-1
```

---

## Scaling Formula

```
desiredReplicas = ceil(messagesVisible / queueLength)
```

With `queueLength: "5"` and `maxReplicaCount: 10`:

| Messages visible | Desired replicas | Actual replicas |
|---|---|---|
| 0 | 0 | 1 (minReplicaCount floor) |
| 1–5 | 1 | 1 |
| 6–10 | 2 | 2 |
| 11–15 | 3 | 3 |
| 46–50 | 10 | 10 |
| 100 | 20 | 10 (maxReplicaCount ceiling) |

---

## Timing

Two independent loops determine how quickly the system reacts:

| Loop | Period | Configurable? |
|---|---|---|
| KEDA polls SQS (`pollingInterval`) | 30s default | Yes — field on `ScaledObject` |
| HPA evaluates the metric | 15s (cluster-wide) | No — control plane flag, not available on managed clusters |

**Worst-case reaction time:** ~45s from a message appearing in the queue to a scale-up decision being made, plus pod startup time.

To tighten the KEDA side:

```yaml
spec:
  pollingInterval: 15   # poll SQS every 15 seconds
```

---

## Cooldown

After the queue drains to zero, KEDA waits `cooldownPeriod` seconds before scaling back down to `minReplicaCount`. This prevents thrash when messages arrive in bursts with short gaps.

```yaml
spec:
  cooldownPeriod: 300   # 5 minutes
```

---

## Additional Trigger Options

```yaml
metadata:
  queueURL: "..."
  queueLength: "5"
  awsRegion: eu-west-1

  # Scale up only when at least this many messages are visible.
  # Prevents a replica spin-up for a single stray message.
  activationQueueLength: "3"

  # Whether in-flight (received but not yet deleted) messages count toward the target.
  # true  → messages being processed count, avoids over-provisioning
  # false → only unprocessed messages count
  scaleOnInFlight: "true"
```

---

## Helm Flags

All KEDA resources in the backend chart are gated behind feature flags:

| Flag | Controls |
|---|---|
| `keda.enabled` + `awsCredentials.enabled` | `TriggerAuthentication` |
| `keda.enabled` + `sqs.enabled` | `ScaledObject` |

Both flags must be `true` for the full setup to render. In practice all three (`keda.enabled`, `awsCredentials.enabled`, `sqs.enabled`) are enabled together in prod.

| Value | Default | Description |
|---|---|---|
| `keda.enabled` | `false` | Master switch |
| `keda.minReplicaCount` | `1` | Minimum replicas (never scale to zero) |
| `keda.maxReplicaCount` | `10` | Hard ceiling on replicas |
| `keda.queueLength` | `"5"` | Target messages per replica |
| `keda.cooldownPeriod` | `300` | Seconds before scaling back down |
