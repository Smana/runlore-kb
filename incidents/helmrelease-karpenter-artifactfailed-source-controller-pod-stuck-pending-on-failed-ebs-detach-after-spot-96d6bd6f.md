---
type: Incident
title: HelmRelease/karpenter ArtifactFailed — source-controller pod stuck Pending on failed EBS detach after spot…
description: 'A Karpenter-managed spot instance in nodepool ''default'' was terminated by AWS at 14:03:16Z (BidEvictedEvent / Server.SpotInstanceTermination). The source-controller pod was running on that node with a ReadWriteOnce gp3 EBS-backed PVC (volumeHandle vol-02e110f306702f13f, PV pvc-e4485222-724e-4456-8d08-81e8855476ad). After the node was drained and the pod evicted (ExceededGracePeriod at 14:05:31Z), the new source-controller pod source-controller-c6b548467-6qk5w was rescheduled to ip-10-0-38-1 at 14:07:07Z but cannot start because the EBS volume is still exclusively attached to the terminated node — FailedAttachVolume: ''Volume is already exclusively attached to one node, waiting on detach before it can be attached to another node''. With source-controller stuck Pending (ready=0/1), the Service source-controller.flux-system.svc.cluster.local has no healthy backend, so the HelmRelease karpenter cannot download its chart artifact and fails with ''context deadline exceeded'' / ArtifactFailed. The OCIRepository kube-system/karpenter itself is healthy (Ready=True, artifact stored 2026-11-07) — the failure is purely the unavailability of the source-controller pod serving the artifact.'
resource: flux-system/source-controller
alert_resource: karpenter/karpenter
tags:
    - runlore
    - incident
    - pod
    - flux-system
timestamp: "2026-11-09T14:10:43Z"
fingerprint: 96d6bd6fc7667ae4ad3b3d487f8c31c0af1192c90ab9ed96a938a992f6ab3097
confidence: 0.88
origin: runlore
---

## Decision

- **why keep:** A Karpenter-managed spot instance in nodepool 'default' was terminated by AWS at 14:03:16Z (BidEvictedEvent / Server.SpotInstanceTermination). The source-controller pod was running on that node with a ReadWriteOnce gp3 EBS-backed PVC (volumeHandle vol-02e110f306702f13f, PV pvc-e4485222-724e-4456-8d08-81e8855476ad). After the node was drained and the pod evicted (ExceededGracePeriod at 14:05:31Z), the new source-controller pod source-controller-c6b548467-6qk5w was rescheduled to ip-10-0-38-1 at 14:07:07Z but cannot start because the EBS volume is still exclusively attached to the terminated node — FailedAttachVolume: 'Volume is already exclusively attached to one node, waiting on detach before it can be attached to another node'. With source-controller stuck Pending (ready=0/1), the Service source-controller.flux-system.svc.cluster.local has no healthy backend, so the HelmRelease karpenter cannot download its chart artifact and fails with 'context deadline exceeded' / ArtifactFailed. The OCIRepository kube-system/karpenter itself is healthy (Ready=True, artifact stored 2026-11-07) — the failure is purely the unavailability of the source-controller pod serving the artifact.
- **confidence:** 88%

## Symptom

HelmRelease/karpenter ArtifactFailed — source-controller pod stuck Pending on failed EBS detach after spot instance termination

Affected resource: Pod flux-system/source-controller-c6b548467-6qk5w

## Investigate

- pod_status flux-system: source-controller-c6b548467-6qk5w Pending ready=0/1 age=33s node=ip-10-0-38-1
- kube_events flux-system source-controller-c6b548467-6qk5w: 'Warning FailedAttachVolume: Waiting for detach for volume pvc-e4485222-724e-4456-8d08-81e8855476ad Volume is already exclusively attached to one node, waiting on detach before it can be attached to another node' at 14:07:07Z
- resource_spec PersistentVolume pvc-e4485222-724e-4456-8d08-81e8855476ad: csi.driver=ebs.csi.aws.com, volumeHandle=vol-02e110f306702f13f, accessModes=[ReadWriteOnce], storageClassName=gp3, nodeAffinity zone=eu-west-3c, phase=Bound
- incident_timeline flux-system: [cloud] ec2.amazonaws.com BidEvictedEvent at 14:03:16Z; [event] Pod/source-controller-c6b548467-nrcld ExceededGracePeriod at 14:05:31Z
- cloud_resource_health: Karpenter nodepool default: instances=6 spot=6 terminated=1 spot-term-reasons=[Server.SpotInstanceTermination]
- gitops_resource_status HelmRelease karpenter/karpenter: Ready=False (ArtifactFailed) — 'Could not load chart: failed to download archive: GET http://source-controller.flux-system.svc.cluster.local./ocirepository/kube-system/karpenter/sha256:5cbfa7669966...tar.gz giving up after 3 attempt(s): context deadline exceeded' at 14:07:29Z
- resource_spec OCIRepository kube-system/karpenter: status.conditions Ready=True (Succeeded), ArtifactInStorage=True, SourceVerified=True — artifact stored 2026-11-07T09:02:30Z, confirming the source object is healthy and the problem is the source-controller pod serving it

## Cause

1. **A Karpenter-managed spot instance in nodepool 'default' was terminated by AWS at 14:03:16Z (BidEvictedEvent / Server.SpotInstanceTermination). The source-controller pod was running on that node with a ReadWriteOnce gp3 EBS-backed PVC (volumeHandle vol-02e110f306702f13f, PV pvc-e4485222-724e-4456-8d08-81e8855476ad). After the node was drained and the pod evicted (ExceededGracePeriod at 14:05:31Z), the new source-controller pod source-controller-c6b548467-6qk5w was rescheduled to ip-10-0-38-1 at 14:07:07Z but cannot start because the EBS volume is still exclusively attached to the terminated node — FailedAttachVolume: 'Volume is already exclusively attached to one node, waiting on detach before it can be attached to another node'. With source-controller stuck Pending (ready=0/1), the Service source-controller.flux-system.svc.cluster.local has no healthy backend, so the HelmRelease karpenter cannot download its chart artifact and fails with 'context deadline exceeded' / ArtifactFailed. The OCIRepository kube-system/karpenter itself is healthy (Ready=True, artifact stored 2026-11-07) — the failure is purely the unavailability of the source-controller pod serving the artifact.** (88%)

## Resolution

- Force-detach the stuck EBS volume vol-02e110f306702f13f in the AWS EC2 console/API (it is still attached to the terminated spot node). Once detached, Kubernetes will attach it to the new source-controller pod on ip-10-0-38-1, the pod will become Ready, and the HelmRelease karpenter will retry its chart download and succeed automatically. If the volume does not detach promptly, verify the old node is fully terminated — AWS force-detaches volumes on instance termination but this can lag several minutes. To prevent recurrence, consider migrating source-controller's PVC to ReadWriteMany (e.g. EFS) or adding a pod anti-affinity / nodepool requirement to keep Flux controllers off spot instances. (reversible=true)

## Unresolved

- Whether placing Flux control-plane controllers (source-controller, etc.) on spot instances is an intentional design choice or should be moved to on-demand instances to prevent recurrence — this is an infrastructure design decision for the platform team

