---
type: Incident
title: '[Pre-existing/recurrent] karpenter-nodepools ArtifactFailed — EKS nodegroup rotation evicted source-controller…'
description: EKS managed-nodegroup node rotation (CompleteLifecycleAction by EKS → TerminateInstances by AutoScaling at 12:17:05 for i-0ed61cfb12dc28953) evicted the source-controller pod. Its RWO EBS PVC (pvc-2fef3d77...) is node-affinity-bound and got stuck with a Multi-Attach error (volume still exclusively attached to the terminating node), causing ~6 min of source-controller downtime. With source-controller unavailable, the ExternalArtifact/infra-artifact source could not be served, and karpenter-nodepools reported ArtifactFailed ('Source is not ready, artifact not found'). Self-healed once the volume detached and the pod rescheduled to ip-10-0-4-218 at 12:17:10.
resource: flux-system/karpenter-nodepools
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-09-08T12:27:24Z"
fingerprint: 07c8594e60e022753f8ff4157b1935708e0d2a45a9d883d23c924a21dcda3ce0
confidence: 0.8
provenance:
    - No GitOps change; EKS nodegroup rotation at 2026-09-08T12:17:04Z (CompleteLifecycleAction by EKS)
---

## Decision

- **why keep:** EKS managed-nodegroup node rotation (CompleteLifecycleAction by EKS → TerminateInstances by AutoScaling at 12:17:05 for i-0ed61cfb12dc28953) evicted the source-controller pod. Its RWO EBS PVC (pvc-2fef3d77...) is node-affinity-bound and got stuck with a Multi-Attach error (volume still exclusively attached to the terminating node), causing ~6 min of source-controller downtime. With source-controller unavailable, the ExternalArtifact/infra-artifact source could not be served, and karpenter-nodepools reported ArtifactFailed ('Source is not ready, artifact not found'). Self-healed once the volume detached and the pod rescheduled to ip-10-0-4-218 at 12:17:10.
- **confidence:** 80%
- **provenance:** No GitOps change; EKS nodegroup rotation at 2026-09-08T12:17:04Z (CompleteLifecycleAction by EKS)

## Symptom

[Pre-existing/recurrent] karpenter-nodepools ArtifactFailed — EKS nodegroup rotation evicted source-controller (PVC Multi-Attach) + Kyverno webhook down; self-healed (occurrence #2)

Affected resource: Kustomization flux-system/karpenter-nodepools

## Investigate

- kube_events: FailedAttachVolume 'Multi-Attach error for volume pvc-2fef3d77-4e9a-415b-966a-0812547e7d06 Volume is already exclusively attached to one node' at 12:16:03Z
- kube_events: FailedScheduling source-controller '0/6 nodes are available: 1 node(s) didn't match PersistentVolume's node affinity, 1 node(s) were unschedulable, 2 Insufficient memory, 3 Insufficient cpu' at 12:16:02Z
- cloud_what_changed: 'TerminateInstances by AutoScaling' for i-0ed61cfb12dc28953 at 12:17:05Z and 'CompleteLifecycleAction by EKS' at 12:17:04Z
- kube_events: SuccessfulAttachVolume at 12:16:21Z, container Started at 12:17:10Z, then karpenter-nodepools ReconciliationSucceeded at 12:19:47Z
- gitops_resource_status: Kustomization karpenter-nodepools Ready=True, sourceRef ExternalArtifact/infra-artifact; event at 11:24:29Z 'DependencyNotReady: Dependencies do not meet ready condition, retrying in 10s'
- query_metrics_range node condition: ip-10-0-16-41 NotReady max=1@12:17:00Z recovered 12:18:00Z; ip-10-0-23-241 NotReady max=1@12:15:00Z
- kube_events: FailedCreate ReplicaSet/flux-operator-589c7d8cb at 12:13:47Z: 'failed calling webhook vpol.validate.kyverno.svc-fail: no endpoints available for service kyverno-svc'
- kube_events: FluxInstance/flux ReconciliationFailed (x5) at 12:14:24Z: 'Deployment/flux-system/helm-controller dry-run failed (InternalError): failed calling webhook vpol.validate.kyverno.svc-fail: no endpoints available for service kyverno-svc'
- kube_events (security ns): kyverno-admission-controller-84bc4cc456-hcnxn Scheduled at 12:13:47Z, containers Started at 12:13:52Z and 12:14:14Z — webhook endpoints restored after startup
- pod_status: kyverno-admission-controller-84bc4cc456-hcnxn Running ready=2/2 age=9m (restarted during incident)

## Cause

1. **EKS managed-nodegroup node rotation (CompleteLifecycleAction by EKS → TerminateInstances by AutoScaling at 12:17:05 for i-0ed61cfb12dc28953) evicted the source-controller pod. Its RWO EBS PVC (pvc-2fef3d77...) is node-affinity-bound and got stuck with a Multi-Attach error (volume still exclusively attached to the terminating node), causing ~6 min of source-controller downtime. With source-controller unavailable, the ExternalArtifact/infra-artifact source could not be served, and karpenter-nodepools reported ArtifactFailed ('Source is not ready, artifact not found'). Self-healed once the volume detached and the pod rescheduled to ip-10-0-4-218 at 12:17:10.** (80%) — change: No GitOps change; EKS nodegroup rotation at 2026-09-08T12:17:04Z (CompleteLifecycleAction by EKS)
2. **Kyverno admission-controller pods were simultaneously evicted by the same node rotation, causing 'no endpoints available for service kyverno-svc' for ~1 min. This compounded the disruption: FluxInstance/flux reconciliation failed (helm-controller dry-run blocked by Kyverno webhook at 12:14:24Z), and flux-operator ReplicaSet creation was rejected (12:13:47Z). Kyverno recovered once its pods rescheduled at 12:13:47-12:14:14Z.** (35%)

## Resolution

- Make source-controller resilient to node rotation: either (a) remove the node-affinity constraint on its PVC / switch to a storage class that allows faster detach-reattach, (b) schedule source-controller on Karpenter provisioned nodes with interruption disruption budgets rather than the EKS managed nodegroup being rotated, or (c) pin source-controller to an on-demand node that is exempt from ASG rotation. The current RWO PVC + node affinity + spot/rotated nodes pattern has now caused 2 ArtifactFailed incidents in ~4h. (reversible=true)
- Ensure Kyverno admission-controller pods have adequate PDB (PodDisruptionBudget) or are scheduled on nodes not subject to simultaneous rotation, so the admission webhook stays available during node churn. Consider failurePolicy: IfNotPresent for the kyverno webhook if appropriate, to avoid blocking critical system namespace operations during Kyverno pod restarts. (reversible=true)

## Unresolved

- The EKS managed-nodegroup rotation at 12:17 was likely a routine update or spot reclamation, but the specific trigger (EKS version update, ASG refresh, or spot interruption) could not be determined from available tools. The 'CompleteLifecycleAction by EKS' event suggests an EKS-managed operation rather than a pure spot interruption.

## Citations

[1] No GitOps change; EKS nodegroup rotation at 2026-09-08T12:17:04Z (CompleteLifecycleAction by EKS)

