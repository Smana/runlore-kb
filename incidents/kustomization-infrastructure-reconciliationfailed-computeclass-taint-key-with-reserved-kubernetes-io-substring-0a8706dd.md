---
type: Incident
title: 'Kustomization/infrastructure ReconciliationFailed: ComputeClass taint key with reserved `kubernetes.io` substring…'
description: A Git change (ExternalArtifact infra-artifact revision 7dd513307c76, synced at ~06:04Z on 2026-10-01) introduced a taint key `ignore-taint.cluster-autoscaler.kubernetes.io/cilium-agent-not-ready` on the NodePoolConfig.Taints of ComputeClass resources (general-purpose, gpu-l4, and io). GKE Warden's `custom-compute-class-limitation` constraint rejects taint/label keys containing the reserved `kubernetes.io` substring, causing the Flux kustomize-controller's server-side dry-run to be denied. The Kustomization was previously succeeding (ReconciliationSucceeded x626 at 06:02:20Z on the prior revision) and began failing immediately upon picking up the new revision.
resource: flux-system/infrastructure
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-10-01T06:10:35Z"
fingerprint: 0a8706dd35132d9a8cf2ab9252dca4c0850f5c1a3cb1adffbcfe6785b05b4ad5
confidence: 0.85
---

## Decision

- **why keep:** A Git change (ExternalArtifact infra-artifact revision 7dd513307c76, synced at ~06:04Z on 2026-10-01) introduced a taint key `ignore-taint.cluster-autoscaler.kubernetes.io/cilium-agent-not-ready` on the NodePoolConfig.Taints of ComputeClass resources (general-purpose, gpu-l4, and io). GKE Warden's `custom-compute-class-limitation` constraint rejects taint/label keys containing the reserved `kubernetes.io` substring, causing the Flux kustomize-controller's server-side dry-run to be denied. The Kustomization was previously succeeding (ReconciliationSucceeded x626 at 06:02:20Z on the prior revision) and began failing immediately upon picking up the new revision.
- **confidence:** 85%

## Symptom

Kustomization/infrastructure ReconciliationFailed: ComputeClass taint key with reserved `kubernetes.io` substring rejected by GKE Warden

Affected resource: Kustomization flux-system/infrastructure

## Investigate

- what_changed: 'flux Kustomization/infrastructure (sync): 12a33760fa09edd9340d681bb8f4e3a5ce2326d25f1344b70910c59f62435051..7dd513307c76bb4d7a03c0de0176bf5e9bab055a3b00f2ed3a92e233e2e585b0 at 2026-10-01T06:05:12Z' — a revision change occurred
- gitops_resource_status: Ready=False (ReconciliationFailed). Events show ReconciliationSucceeded(x626) at 06:02:20Z followed by ReconciliationFailed starting at 06:04:23Z — the failure began immediately after the new revision was picked up
- kube_events: three ComputeClasses failing with the identical taint key: 'compute-class "general-purpose" NodePoolConfig.Taints with invalid key "ignore-taint.cluster-autoscaler.kubernetes.io/cilium-agent-not-ready": label/taint key cannot contain reserved `kubernetes.io` substring' (same for gpu-l4 at 06:08:16Z and io at 06:07:14Z)
- controller_logs (kustomize-controller): at 06:04:16Z 'Reconciliation failed ... error: ComputeClass/general-purpose dry-run failed (GKE Warden constraints violations): admission webhook "warden-validating.common-webhooks.networking.gke.io" denied the request' with revision 'latest@sha256:7dd513307c76bb4d7a03c0de0176bf5e9bab055a3b00f2ed3a92e233e2e585b0' — the same revision that what_changed reports as the new target
- gitops_tree: the dependency chain is healthy — namespaces (Ready=True), crossplane-configuration (Ready=True), source ExternalArtifact infra-artifact (Ready=True). The failure is isolated to the infrastructure Kustomization's own manifests, not an upstream dependency
- cloud_what_changed: no GKE Warden or constraint policy changes in the audit logs — only routine Crossplane provider and Kyverno report operations. This rules out a newly-enabled Warden constraint as the cause

## Cause

1. **A Git change (ExternalArtifact infra-artifact revision 7dd513307c76, synced at ~06:04Z on 2026-10-01) introduced a taint key `ignore-taint.cluster-autoscaler.kubernetes.io/cilium-agent-not-ready` on the NodePoolConfig.Taints of ComputeClass resources (general-purpose, gpu-l4, and io). GKE Warden's `custom-compute-class-limitation` constraint rejects taint/label keys containing the reserved `kubernetes.io` substring, causing the Flux kustomize-controller's server-side dry-run to be denied. The Kustomization was previously succeeding (ReconciliationSucceeded x626 at 06:02:20Z on the prior revision) and began failing immediately upon picking up the new revision.** (85%)

## Resolution

- Fix forward: rename or remove the taint key `ignore-taint.cluster-autoscaler.kubernetes.io/cilium-agent-not-ready` in the ComputeClass manifests (general-purpose, gpu-l4, io) so it does not contain the reserved `kubernetes.io` substring — e.g. use a custom domain prefix like `ignore-taint.cluster-autoscaler.runlore.io/cilium-agent-not-ready`. Then push the fix and let Flux reconcile. Alternatively, revert to the prior known-good revision (12a33760fa09) to immediately restore reconciliation while the taint key is corrected. To stop reconciliation churn in the interim, `flux suspend kustomization infrastructure`. (reversible=true)

