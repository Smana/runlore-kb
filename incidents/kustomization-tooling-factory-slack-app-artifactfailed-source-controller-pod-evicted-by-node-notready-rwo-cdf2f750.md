---
type: Incident
title: Kustomization/tooling-factory-slack-app ArtifactFailed — source-controller pod evicted by node NotReady, RWO…
description: 'Node ip-10-0-6-27.eu-west-3.compute.internal went NotReady at 2026-11-18T16:20:00Z, evicting the source-controller pod (source-controller-c6b548467-vpnsz) that was scheduled on it. The source-controller serves OCIRepository artifacts to Kustomizations via its ClusterIP Service (172.20.111.37:80). With the pod down, the kustomize-controller could not download the ai-native-ref artifact (sha256:db5bb6bebb25...), resulting in the ArtifactFailed ''dial tcp 172.20.111.37:80: i/o timeout'' after 10 retries. This is a recurrence of the documented pattern: source-controller''s RWO EBS PVC (pvc-e33564c4-dd5b-4c65-be51-0e3a3bbc5b13, volume vol-0e694abbcf07a307d, storageClass gp3) has nodeAffinity bound to zone eu-west-3a and the Deployment uses strategy: Recreate, so the pod cannot reschedule until the EBS volume detaches from the dead node and reattaches to a new node in the same zone.'
resource: flux-system/source-controller
alert_resource: flux-system/tooling-factory-slack-app
tags:
    - runlore
    - incident
    - persistentvolumeclaim
    - flux-system
timestamp: "2026-11-18T17:59:19Z"
fingerprint: cdf2f7503d75023cd7878a9081e20c152f4b260c6dd5c464683f7bcdb47dde42
confidence: 0.78
origin: runlore
---

## Decision

- **why keep:** Node ip-10-0-6-27.eu-west-3.compute.internal went NotReady at 2026-11-18T16:20:00Z, evicting the source-controller pod (source-controller-c6b548467-vpnsz) that was scheduled on it. The source-controller serves OCIRepository artifacts to Kustomizations via its ClusterIP Service (172.20.111.37:80). With the pod down, the kustomize-controller could not download the ai-native-ref artifact (sha256:db5bb6bebb25...), resulting in the ArtifactFailed 'dial tcp 172.20.111.37:80: i/o timeout' after 10 retries. This is a recurrence of the documented pattern: source-controller's RWO EBS PVC (pvc-e33564c4-dd5b-4c65-be51-0e3a3bbc5b13, volume vol-0e694abbcf07a307d, storageClass gp3) has nodeAffinity bound to zone eu-west-3a and the Deployment uses strategy: Recreate, so the pod cannot reschedule until the EBS volume detaches from the dead node and reattaches to a new node in the same zone.
- **confidence:** 78%

## Symptom

Kustomization/tooling-factory-slack-app ArtifactFailed — source-controller pod evicted by node NotReady, RWO zone-affinity EBS PVC delayed rescheduling ~9 min (self-healed)

Affected resource: PersistentVolumeClaim flux-system/source-controller

## Investigate

- query_metrics_range kube_node_status_condition{Ready,status='true',node='ip-10-0-6-27'}: trend=1>1>...>0, biggest jump -1 at 2026-11-18T16:20:00Z — node went NotReady at 16:20Z
- query_metrics_range kube_pod_info{pod=~'source-controller-c6b548467.*'}: source-controller-c6b548467-vpnsz was on ip-10-0-6-27 (the failed node); source-controller-c6b548467-h5zfz appeared on ip-10-0-11-129 at 16:24:00Z (new pod)
- resource_spec PersistentVolume pvc-e33564c4-...: accessModes=[ReadWriteOnce], storageClassName=gp3, csi.driver=ebs.csi.aws.com, volumeHandle=vol-0e694abbcf07a307d, nodeAffinity required zone=eu-west-3a
- resource_spec Deployment source-controller: strategy.type=Recreate, volume persistent-data uses PVC source-controller
- cloud_what_changed for vol-0e694abbcf07a307d: DetachVolume at 16:29:03Z (FAILED: Client.IncorrectState 'Volume is in available state'), AttachVolume at 16:29:04Z — volume reattach completed ~9 minutes after node failure
- resource_spec Deployment source-controller status: Available condition lastTransitionTime=2026-11-18T16:29:12Z — pod became available 9 minutes after node went NotReady at 16:20:00Z
- gitops_resource_status Kustomization tooling-factory-slack-app: Ready=True (ReconciliationSucceeded), Applied revision sha256:db5bb6bebb25...; OCIRepository ai-native-ref: Ready=True — both self-healed after source-controller recovered
- kb_search returned matching runbook: 'EKS managed-nodegroup node rotation evicted the source-controller pod. Its RWO EBS PVC is node-affinity-bound and got stuck... causing ~6 min of source-controller downtime... karpenter-nodepools reported ArtifactFailed (Source is not ready, artifact not found). Self-healed once the volume detached and the pod rescheduled'
- cloud_what_changed (180m window): no TerminateInstances or CompleteLifecycleAction events found; only SSM UpdateInstanceInformation and ec2 DryRunOperation failures
- cloud_resource_health i-0ed61cfb12dc28953 (from past incident): InvalidInstanceID.NotFound — the instance from the previous incident no longer exists
- pod_status flux-system: ip-10-0-6-27 does not appear among current nodes; source-controller now runs on ip-10-0-11-129

## Cause

1. **Node ip-10-0-6-27.eu-west-3.compute.internal went NotReady at 2026-11-18T16:20:00Z, evicting the source-controller pod (source-controller-c6b548467-vpnsz) that was scheduled on it. The source-controller serves OCIRepository artifacts to Kustomizations via its ClusterIP Service (172.20.111.37:80). With the pod down, the kustomize-controller could not download the ai-native-ref artifact (sha256:db5bb6bebb25...), resulting in the ArtifactFailed 'dial tcp 172.20.111.37:80: i/o timeout' after 10 retries. This is a recurrence of the documented pattern: source-controller's RWO EBS PVC (pvc-e33564c4-dd5b-4c65-be51-0e3a3bbc5b13, volume vol-0e694abbcf07a307d, storageClass gp3) has nodeAffinity bound to zone eu-west-3a and the Deployment uses strategy: Recreate, so the pod cannot reschedule until the EBS volume detaches from the dead node and reattaches to a new node in the same zone.** (78%)
2. **The specific trigger for node ip-10-0-6-27 going NotReady at 16:20:00Z could not be determined from available tools. No TerminateInstances, CompleteLifecycleAction, or ASG scaling activity events appeared in CloudTrail within the 180-minute window — only routine SSM UpdateInstanceInformation heartbeats and some DryRunOperation failures (non-impacting). The node is no longer in the cluster (it does not appear in the current node list from pod_status). This is consistent with an EKS-managed node rotation or spot interruption, but the exact cause (EKS version update, ASG refresh, spot reclamation) was not visible in CloudTrail.** (20%)

## Resolution

- Make source-controller resilient to node rotation, as the KB runbook recommends (this is now the 3rd+ occurrence): (a) switch the source-controller PVC to a storage class that allows faster detach/reattach, or remove the node-affinity constraint; (b) schedule source-controller on a Karpenter-provisioned node with disruption budgets rather than the EKS managed nodegroup being rotated; or (c) pin source-controller to an on-demand node exempt from ASG rotation. The RWO PVC + node affinity + Recreate strategy pattern has now caused multiple ArtifactFailed incidents. (reversible=true)

## Unresolved

- The specific trigger for node ip-10-0-6-27 going NotReady at 16:20:00Z was not found in CloudTrail — no TerminateInstances/CompleteLifecycleAction/ASG scaling events appeared. It may have been a spot interruption, EKS-managed update, or ASG refresh that is not visible in the CloudTrail event types captured by cloud_what_changed. A human should check AWS Console or CloudWatch for spot interruption notices or ASG activity around 16:20Z.

