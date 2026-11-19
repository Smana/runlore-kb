---
type: Incident
title: '[Pre-existing] HelmRelease/security/cert-manager ArtifactFailed — source-controller RWO PVC Multi-Attach during…'
description: 'A Karpenter spot node was terminated (Server.SpotInstanceTermination), forcing the source-controller pod (flux-system/source-controller-c6b548467-2psqx) to reschedule to a new node. The source-controller uses a ReadWriteOnce PVC (pvc-e33564c4-dd5b-4c65-be51-0e3a3bbc5b13) for chart artifact storage, which can only be attached to one node at a time. The previous pod (h5zfz) did not terminate cleanly (ExceededGracePeriod at 09:45:39Z), so the PVC remained attached to the dying node. The new pod, scheduled at 09:47:08Z, hit FailedAttachVolume: ''Volume is already exclusively attached to one node, waiting on detach before it can be attached to another node'' — blocking pod startup for ~6 minutes (09:47→09:53). During this window, the helm-controller could not download the cert-manager chart from the unavailable source-controller, producing the ArtifactFailed ''context deadline exceeded'' errors at 09:46:53Z and 09:52:46Z. This is the SAME root cause as the previous investigation 17h ago — a recurring architectural vulnerability: RWO PVC on spot nodes.'
resource: flux-system/source-controller
alert_resource: security/cert-manager
tags:
    - runlore
    - incident
    - pod
    - flux-system
timestamp: "2026-11-19T10:10:54Z"
fingerprint: 9ce7bbd6886cf854dae23be9572f2b9a0449e37cc24728d2589d88f1201b9a0d
confidence: 0.85
provenance:
    - No GitOps change — Karpenter spot node termination (Server.SpotInstanceTermination) triggered the pod reschedule. what_changed(security/cert-manager) found no GitOps object; incident_timeline showed only unrelated Kustomization syncs (crds-external-secrets, crds-kyverno).
origin: runlore
---

## Decision

- **why keep:** A Karpenter spot node was terminated (Server.SpotInstanceTermination), forcing the source-controller pod (flux-system/source-controller-c6b548467-2psqx) to reschedule to a new node. The source-controller uses a ReadWriteOnce PVC (pvc-e33564c4-dd5b-4c65-be51-0e3a3bbc5b13) for chart artifact storage, which can only be attached to one node at a time. The previous pod (h5zfz) did not terminate cleanly (ExceededGracePeriod at 09:45:39Z), so the PVC remained attached to the dying node. The new pod, scheduled at 09:47:08Z, hit FailedAttachVolume: 'Volume is already exclusively attached to one node, waiting on detach before it can be attached to another node' — blocking pod startup for ~6 minutes (09:47→09:53). During this window, the helm-controller could not download the cert-manager chart from the unavailable source-controller, producing the ArtifactFailed 'context deadline exceeded' errors at 09:46:53Z and 09:52:46Z. This is the SAME root cause as the previous investigation 17h ago — a recurring architectural vulnerability: RWO PVC on spot nodes.
- **confidence:** 85%
- **provenance:** No GitOps change — Karpenter spot node termination (Server.SpotInstanceTermination) triggered the pod reschedule. what_changed(security/cert-manager) found no GitOps object; incident_timeline showed only unrelated Kustomization syncs (crds-external-secrets, crds-kyverno).

## Symptom

[Pre-existing] HelmRelease/security/cert-manager ArtifactFailed — source-controller RWO PVC Multi-Attach during Karpenter spot node termination (recurrence #2, self-recovered)

Affected resource: Pod flux-system/source-controller-c6b548467-2psqx

## Investigate

- kube_events (flux-system): 09:47:08Z Warning FailedAttachVolume: 'Waiting for detach for volume pvc-e33564c4-dd5b-4c65-be51-0e3a3bbc5b13 Volume is already exclusively attached to one node, waiting on detach before it can be attached to another node'
- kube_events (flux-system): 09:45:39Z Warning ExceededGracePeriod on source-controller-h5zfz: 'Container runtime did not kill the pod within specified grace period' — old pod didn't terminate cleanly, keeping PVC attached
- cloud_resource_health: 'Karpenter nodepool default: instances=6 spot=6 terminated=1 spot-term-reasons=[Server.SpotInstanceTermination]' — a spot node was terminated
- gitops_resource_status HelmRelease/security/cert-manager: Ready=True (InstallSucceeded) at 09:53:33Z, with Warning ArtifactFailed events at 09:46:53Z and 09:52:46Z: 'failed to download archive: context deadline exceeded' and 'GET http://source-controller.flux-system.../cert-manager-v1.21.3.tgz giving up after 2 attempt(s): context deadline exceeded'
- kube_events (flux-system): 09:53:13Z Normal SuccessfulAttachVolume for pvc-e33564c4-dd5b-4c65-be51-0e3a3bbc5b13, followed by 09:53:17Z Container Started — pod recovered after ~6 min delay
- controller_logs (source-controller, cert-manager): from 09:53:17Z onward, 'artifact up-to-date with remote revision: v1.21.3' reconciliations resumed normally — source-controller was healthy after PVC reattachment
- kube_events (flux-system): 09:41:21Z Warning Kustomization/zitadel ReconciliationFailed: 'Certificate/security/zitadel dry-run failed (InternalError): failed calling webhook webhook.cert-manager.io: no endpoints available for service cert-manager-webhook'
- pod_status (security): cert-manager-webhook-57f565cf59-5wwkn age=27m (recently rescheduled), now Running ready=1/1 — confirms the webhook pod was disrupted and recovered
- pod_status (security): cert-manager-cainjector-76f6485b5b-45srb age=1h (also recently rescheduled) — multiple security pods were affected by node churn

## Cause

1. **A Karpenter spot node was terminated (Server.SpotInstanceTermination), forcing the source-controller pod (flux-system/source-controller-c6b548467-2psqx) to reschedule to a new node. The source-controller uses a ReadWriteOnce PVC (pvc-e33564c4-dd5b-4c65-be51-0e3a3bbc5b13) for chart artifact storage, which can only be attached to one node at a time. The previous pod (h5zfz) did not terminate cleanly (ExceededGracePeriod at 09:45:39Z), so the PVC remained attached to the dying node. The new pod, scheduled at 09:47:08Z, hit FailedAttachVolume: 'Volume is already exclusively attached to one node, waiting on detach before it can be attached to another node' — blocking pod startup for ~6 minutes (09:47→09:53). During this window, the helm-controller could not download the cert-manager chart from the unavailable source-controller, producing the ArtifactFailed 'context deadline exceeded' errors at 09:46:53Z and 09:52:46Z. This is the SAME root cause as the previous investigation 17h ago — a recurring architectural vulnerability: RWO PVC on spot nodes.** (85%) — change: No GitOps change — Karpenter spot node termination (Server.SpotInstanceTermination) triggered the pod reschedule. what_changed(security/cert-manager) found no GitOps object; incident_timeline showed only unrelated Kustomization syncs (crds-external-secrets, crds-kyverno).
2. **Downstream impact: the same spot node termination also disrupted cert-manager-webhook (pod only 27m old at investigation time, recently rescheduled). At 09:41:21Z, cert-manager-webhook had no available endpoints, causing Kustomization/flux-system/zitadel ReconciliationFailed: 'Certificate/security/zitadel dry-run failed: failed calling webhook webhook.cert-manager.io: no endpoints available for service cert-manager-webhook'. This is a parallel consequence of the spot node churn, not caused by the source-controller PVC issue itself, but it compounds the blast radius of spot node terminations hitting the security namespace.** (40%)

## Resolution

- No action suggested by the investigation — see `## Unresolved`.

## Unresolved

- Which specific Karpenter node was terminated and whether the source-controller and cert-manager-webhook pods were co-located on it — the cloud_what_changed output was truncated at 25 events (mostly SSM UpdateInstanceInformation noise) and did not clearly identify the terminated instance ID. cloud_resource_health confirms 'terminated=1' but not which instance.

## Citations

[1] No GitOps change — Karpenter spot node termination (Server.SpotInstanceTermination) triggered the pod reschedule. what_changed(security/cert-manager) found no GitOps object; incident_timeline showed only unrelated Kustomization syncs (crds-external-secrets, crds-kyverno).

