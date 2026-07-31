# Index

Table of contents for every subject doc in this repo, with its subsections.
Kept in sync whenever a doc is created or a section is added — see
`CLAUDE.md` for how docs get written.

- [`application_configuration.md`](./application_configuration.md) — Application Configuration: Delivery Mechanisms and Mutability
  - Can Config Be Applied Without Recreating the Pod?
  - Why Recreation Is the Default: Pod Spec Immutability
  - Patterns Mature Teams Use for Config Hot-Reload
  - Takeaway

- [`apply_models.md`](./apply_models.md) — Kubernetes Apply Models: Client-Side vs Server-Side Apply
  - Overview
  - Client-Side Apply
  - Server-Side Apply (SSA)
  - Key Difference
  - Real-World Usage
  - ArgoCD: Two Separate "Server-Side" Settings
  - One-Liners

- [`argocd.md`](./argocd.md) — ArgoCD
  - 1. What ArgoCD Is
  - 2. What a Standard ArgoCD Helm Install Deploys
  - 3. The Application Resource Tree — How Managed Resources Are Discovered
  - 4. Deep Dive: The Self-Heal / Drift-Detection Flow
  - 5. Application Health

- [`argocd_refresh_sync.md`](./argocd_refresh_sync.md) — ArgoCD: Refresh vs Sync
  - Core Loop
  - Refresh
  - Sync
  - Refresh → Sync Flow
  - Annotations as Control Signals
  - Mental Model
  - Best Practices

- [`autoscaling.md`](./autoscaling.md) — Autoscaling: HPA and the Metrics API
  - Is the HorizontalPodAutoscaler a Built-In Controller?
  - How the HPA Controller Actually Reconciles
  - The Metrics API: Aggregation, Not a Built-In Type
  - The Full Chain
  - The Custom Metrics API: What Problem It Solves
  - A Tempting Shortcut: Pointing the Aggregation Layer at Prometheus Directly
  - Prometheus Adapter: Its Exact Role
  - One Rule Per Metric, One Adapter Per Cluster
  - A Third API Group: Why External Metrics Can't Just Be Custom Metrics
  - KEDA: An Operator That Manages HPAs, Not a Metrics Backend That Replaces Them
  - Walking Through a ScaledObject, End to End
  - What This Covers So Far

- [`cadvisor_metrics.md`](./cadvisor_metrics.md) — cAdvisor Metrics, the Pause Container, and Pod-Level Aggregates
  - Overview
  - cAdvisor and its Role
  - The Three Layers cAdvisor Measures
  - How the Filters Work Together
  - Summary Table

- [`concepts.md`](./concepts.md) — Kubernetes Concepts — Quick Reference
  - Deployment Strategy: RollingUpdate (default)
  - Probes
  - Graceful Shutdown
  - External Secrets Operator (ESO) + AWS Secrets Manager

- [`crossplane_architecture.md`](./crossplane_architecture.md) — Crossplane Architecture
  - What is Crossplane
  - Layer 1 — Crossplane Core (the operator)
  - Layer 2 — The Provider (e.g. provider-aws-sqs)
  - Layer 3 — The ProviderConfig
  - Layer 4 — The credentials Secret
  - Full dependency chain
  - Install ordering in this project
  - Why the ProviderConfig is a separate ArgoCD app
  - Compositions and XRDs — The Infrastructure API Layer
  - Drift Detection and Correction
  - Composition Field Reference
  - Composition Functions and Pipeline Mode

- [`gitops_and_argocd.md`](./gitops_and_argocd.md) — GitOps & ArgoCD
  - Q&A format (older doc, predates the generic-file/subsection convention below); overlaps with `argocd.md` and `argocd_refresh_sync.md`. Topics covered: the GitOps model, ArgoCD's role, the reconciliation loop, `OperationState`, tracking/stopping a running sync, ArgoCD vs Jenkins/Tekton, handling manual prod changes, ArgoCD's architecture components, multi-cluster/multi-environment setup, `argocd cluster add` internals and security implications, custom resource health checks via Lua.

- [`go_routines.md`](./go_routines.md) — Channels and Select in Go
  - What is a Channel?
  - Unbuffered Channels
  - How `select` Works
  - The Fibonacci Code
  - Step-by-Step Execution
  - Why Unbuffered Channels Make This Work
  - Reading from a Closed Channel and the `ok` Idiom
  - sync.Once
  - Fan-Out / Fan-In Pipeline Pattern

- [`keda_sqs_autoscaling.md`](./keda_sqs_autoscaling.md) — KEDA SQS Autoscaling
  - What is KEDA?
  - Architecture
  - Credential Flow
  - Resources
  - Scaling Formula
  - Timing
  - Cooldown
  - Additional Trigger Options
  - Helm Flags

- [`keda_webhook_stale_cache.md`](./keda_webhook_stale_cache.md) — KEDA Admission Webhook — Stale Cache After HPA Deletion
  - The Error
  - Root Cause
  - The Second Problem — Stale Webhook Cache
  - Key Takeaways

- [`kubelet.md`](./kubelet.md) — Kubelet & Node-Level Mechanics
  - `kubectl exec` Internals: Namespace-Joining, Proxying, and the Kubelet's Own API

- [`kubernetes_operators.md`](./kubernetes_operators.md) — Kubernetes Operators
  - Chapter 1: From `kubectl apply` to Running Pods
  - Chapter 2: The CRD and Its Role
  - Chapter 3: resourceVersion and Optimistic Concurrency
  - Chapter 4: generation and observedGeneration
  - Chapter 5: Finalizers and Controlled Deletion
  - Chapter 6: Deep Copy and the Cache Protection Problem
  - Chapter 7: Kubebuilder Markers
  - Chapter 8: Resource Structure and Status

- [`kubernetes_scenarios.md`](./kubernetes_scenarios.md) — Kubernetes Scenarios
  - Scenario: Replace an Unresponsive Worker Node
  - Scenario: Ensure Data Persistence for a Stateful Application
  - Scenario: Pod Stuck in Pending State

- [`limit_ranges.md`](./limit_ranges.md) — Kubernetes LimitRange
  - Overview
  - Structure
  - Field Reference
  - Behavior Examples
  - Interactions
  - Environment Design Reference
  - Common Pitfalls
  - Mental Model

- [`logging.md`](./logging.md) — Application Logging: Collection, stdout, and the Risk of Loss
  - Where stdout Actually Goes
  - What `kubectl logs` Actually Does
  - Collection Architecture Patterns
  - Worked Example: Fluent Bit → Loki → Grafana
  - Is There a Risk of Losing Logs?
  - Key Interactions

- [`pipeline_pattern.md`](./pipeline_pattern.md) — Pipeline Pattern in Go
  - What is a Pipeline?
  - A Simple Pipeline
  - Closing Channels and `range`
  - Fan-Out and Fan-In
  - Cancellation with `context`
  - Key Rules
  - Relation to This Codebase

- [`pod_lifecycle_and_restarts.md`](./pod_lifecycle_and_restarts.md) — Pod & Container Lifecycle: Failures and Restarts
  - OOM Kills: Container Restart or Pod Recreation?
  - Readiness vs Liveness: Why Only One of Them Can Trigger a Restart
  - startupProbe and the No-Probe Defaults

- [`pv_and_pvcs.md`](./pv_and_pvcs.md) — Kubernetes Persistent Volumes (PV) & Persistent Volume Claims (PVC)
  - Core Concepts
  - Binding Model (1:1 Relationship)
  - Binding Process (Controller Behavior)
  - Provisioning Types
  - Reclaim Policies
  - PV Lifecycle States
  - Reusing a PV with `reclaimPolicy: Retain`
  - Pod ↔ PVC ↔ PV Dependency and Pending States
    - Scheduler and Volume Binding (`volumeBindingMode`, the `Immediate` zone-affinity pitfall)
    - Finalizers
  - Access Modes: `ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany`, `ReadWriteOncePod`
  - Dynamic Provisioning Walkthrough: A GP3 PVC, End to End
  - Static Provisioning Walkthrough: A Local PV, `WaitForFirstConsumer`
  - Mental Model
  - DevOps Takeaways
  - What This Covers So Far

- [`requests_vs_limits.md`](./requests_vs_limits.md) — Kubernetes: Requests vs Limits
  - Definitions
  - Runtime Behavior
  - What Happens Without Them
  - QoS Classes
  - Autoscaling Implications (HPA / KEDA)
  - LimitRange and ResourceQuota Interaction
  - Best Practices

- [`resource_quotas.md`](./resource_quotas.md) — Resource Quotas
  - What a ResourceQuota actually is
  - What can you limit?
  - Example: Basic ResourceQuota
  - How Kubernetes enforces it
  - Important gotcha (very common)
  - ResourceQuota + LimitRange (critical combo)
  - Common use cases
  - What ResourceQuota does NOT do
  - Quota for specific resource types

- [`scheduling.md`](./scheduling.md) — Kubernetes Scheduling, Preemption, and Eviction
  - Scheduling
    - Binding: Making the Decision Permanent
  - Preemption
  - Eviction
    - PodDisruptionBudget: How "Voluntary" Is Actually Checked
  - Pod Garbage Collection
  - Key Interactions
  - What This Covers So Far

- [`services_and_load_balancing.md`](./services_and_load_balancing.md) — Services & Load Balancing: Spread vs. Balance
  - Can a ClusterIP Service Ensure Load Balancing for TCP Traffic?
  - The Proper Way: An L7-Aware Entity
  - How L7 Entities Actually Achieve Balance
  - Takeaway

- [`sqs_worker_context_cancellation.md`](./sqs_worker_context_cancellation.md) — SQS Worker — Context, Cancellation and Shutdown
  - Goroutines are not child processes
  - Where the worker halts and waits
  - Context
  - Shutdown sequence
  - How each operation handles cancellation
  - The duplicate task edge case
  - SQS cost

- [`workloads.md`](./workloads.md) — Workload Controllers: Deployment, StatefulSet, and DaemonSet
  - Deployment: Fungible Pods, No Identity
    - Update Flow
    - Configuring Zero-Downtime Rollouts
    - Rollback Flow
    - Rollback History
    - Update/Rollback Summary
    - How the ReplicaSet Controller Actually Reconciles
    - Why the ReplicaSet Doesn't Enforce Template Conformance on Existing Pods
    - Avoiding Overshoot: the Expectations Mechanism
    - What Triggers Pod Replacement (and What Doesn't)
    - Graceful Termination During Rollout: preStop and `terminationGracePeriodSeconds`
    - Detecting a Stuck Rollout: `progressDeadlineSeconds`
    - Argo Rollouts: Closing the Observation Gap
    - Rolling Update: What to Watch For
  - StatefulSet: Stable Identity for Clustered / Data-Bearing Workloads
    - Pod identity
    - Storage identity
    - Network identity
    - Worked example
    - Startup/shutdown ordering
  - DaemonSet: One Pod Per Node, Not a Replica Count
    - The reconcile model
    - How a Pod actually lands on its node
    - Taints don't get bypassed automatically
    - Cordon and drain
    - Node deletion
  - The Decision Rule
  - What This Covers So Far
