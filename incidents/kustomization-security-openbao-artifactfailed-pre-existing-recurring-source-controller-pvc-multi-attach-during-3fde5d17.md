---
type: Incident
title: Kustomization/security-openbao ArtifactFailed — pre-existing recurring source-controller PVC Multi-Attach during…
description: 'Recurring (occurrence #2) source-controller PVC Multi-Attach failure during Karpenter spot node termination — source-controller uses a PersistentVolume (pvc-2fef3d77-4e9a-415b-966a-0812547e7d06) with node affinity that cannot be multi-attached to a replacement node. When the spot node hosting source-controller is terminated, the pod fails scheduling (PV node affinity mismatch + Multi-Attach error), causing transient ArtifactFailed/DependencyNotReady that cascades to dependent Kustomizations like security-openbao. Each episode self-resolves once the PVC detaches from the dead node and source-controller reschedules, but the same vulnerability recurs on every spot node termination. This is the same fault mechanism identified 3h32m ago — not a new fault.'
resource: flux-system/security-openbao
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-09-08T12:31:22Z"
fingerprint: 3fde5d173b411f9c82b426d5fc6f021492124196f2979fd0c554805d02a8eb6d
confidence: 0.85
provenance:
    - No Git/config change caused this incident — triggered by cloud infra (Karpenter spot node termination)
---

## Decision

- **why keep:** Recurring (occurrence #2) source-controller PVC Multi-Attach failure during Karpenter spot node termination — source-controller uses a PersistentVolume (pvc-2fef3d77-4e9a-415b-966a-0812547e7d06) with node affinity that cannot be multi-attached to a replacement node. When the spot node hosting source-controller is terminated, the pod fails scheduling (PV node affinity mismatch + Multi-Attach error), causing transient ArtifactFailed/DependencyNotReady that cascades to dependent Kustomizations like security-openbao. Each episode self-resolves once the PVC detaches from the dead node and source-controller reschedules, but the same vulnerability recurs on every spot node termination. This is the same fault mechanism identified 3h32m ago — not a new fault.
- **confidence:** 85%
- **provenance:** No Git/config change caused this incident — triggered by cloud infra (Karpenter spot node termination)

## Symptom

Kustomization/security-openbao ArtifactFailed — pre-existing recurring source-controller PVC Multi-Attach during spot node termination (self-resolved, occurrence #2)

Affected resource: Kustomization flux-system/security-openbao

## Investigate

- kube_events (flux-system): 12:16:02Z Pod/source-controller-6f7cf8cb76-v5qhf FailedScheduling: '0/6 nodes are available: 1 node(s) didn't match PersistentVolume's node affinity, 1 node(s) were unschedulable, 3 Insufficient cpu, 3 Insufficient memory'
- kube_events (flux-system): 12:16:03Z Pod/source-controller-6f7cf8cb76-v5qhf FailedAttachVolume: 'Multi-Attach error for volume pvc-2fef3d77-4e9a-415b-966a-0812547e7d06 Volume is already exclusively attached to one node and can't be attached to another'
- incident_timeline: 12:17:10Z source-controller pod readiness probe failed, then restarted (fresh startup logs in controller_logs at 12:17:10Z showing 'starting manager', 'Attempting to acquire leader lease')
- incident_timeline: 12:18:08Z Kustomization/security-openbao DependencyNotReady (x12): 'Dependencies do not meet ready condition, retrying in 10s'
- gitops_resource_status Kustomization/security-openbao: now Ready=True, 'Applied revision: latest@sha256:fb9178bfa612...' — recovered at 12:26:20Z ReconciliationSucceeded (x32)
- gitops_resource_status ExternalArtifact/security-artifact: Ready=True, 'Artifact is ready' — source is healthy now
- pod_status: source-controller-6f7cf8cb76-v5qhf Running 1/1 age=12m (restarted after the disruption) at 12:17
- cloud_resource_health: 'Karpenter nodepool default: instances=5 spot=5 terminated=1' — confirms a spot node was terminated
- cloud timeline (incident_timeline): 12:18:00Z ec2.amazonaws.com DeleteLaunchTemplate by karpenter — spot node teardown
- kube_events: cascading scheduling failures across kustomize-controller-apps, notification-controller, source-watcher (all FailedScheduling or Unhealthy readiness probes) — consistent with spot node loss reducing cluster capacity
- Previous investigation (3h32m ago): same conclusion — 'transient source-controller PVC Multi-Attach during spot node termination (self-resolved)'

## Cause

1. **Recurring (occurrence #2) source-controller PVC Multi-Attach failure during Karpenter spot node termination — source-controller uses a PersistentVolume (pvc-2fef3d77-4e9a-415b-966a-0812547e7d06) with node affinity that cannot be multi-attached to a replacement node. When the spot node hosting source-controller is terminated, the pod fails scheduling (PV node affinity mismatch + Multi-Attach error), causing transient ArtifactFailed/DependencyNotReady that cascades to dependent Kustomizations like security-openbao. Each episode self-resolves once the PVC detaches from the dead node and source-controller reschedules, but the same vulnerability recurs on every spot node termination. This is the same fault mechanism identified 3h32m ago — not a new fault.** (85%) — change: No Git/config change caused this incident — triggered by cloud infra (Karpenter spot node termination)

## Resolution

- The immediate alert has self-resolved. However, since this is occurrence #2 of the same fault within ~4 hours, consider a structural fix to prevent recurrence: (1) schedule source-controller on an on-demand (non-spot) node via nodeSelector/nodeAffinity or a Karpenter nodepool requirement, or (2) migrate source-controller's PVC to a ReadWriteMany storage class (e.g., EFS) that supports multi-attach across nodes, eliminating the single-node PV attachment constraint. Option 1 is lower-risk and more targeted. (reversible=false)

## Unresolved

- Why source-controller runs with a node-affinity-bound PVC on spot nodes in the first place — this is a cluster design choice outside the scope of available tools; a human should review the source-controller HelmRelease/PVC configuration in Git to determine whether the node affinity is intentional or can be relaxed

## Citations

[1] No Git/config change caused this incident — triggered by cloud infra (Karpenter spot node termination)

