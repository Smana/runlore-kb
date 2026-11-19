---
type: Incident
title: HelmRelease/crossplane ArtifactFailed — Karpenter spot node termination evicted source-controller (RWO EBS PVC…
description: 'A Karpenter spot node was terminated (Server.SpotInstanceTermination) in the ''default'' nodepool, evicting the source-controller pod. Its RWO EBS PVC (pvc-e4485222-724e-4456-8d08-81e8855476ad / vol-02e110f306702f13f, gp3 5Gi) was exclusively attached to the terminating node, so when the pod rescheduled to ip-10-0-38-1 at 14:07:07Z it hit ''FailedAttachVolume: Volume is already exclusively attached to one node, waiting on detach''. The source-controller pod stayed ContainerCreating/Pending for ~6 minutes (14:07:07Z→14:13:12Z), during which the source-controller.flux-system.svc.cluster.local Service (clusterIP 172.20.180.33) had no backing pods. helm-controller could not download the crossplane-2.4.3.tgz chart archive from source-controller, progressing through ''no such host'' → ''i/o timeout'' → ''context deadline exceeded'' errors, producing the HelmRelease/crossplane ArtifactFailed. The source-controller has now recovered (EBS volume attached at 14:13:12Z, pod Running ready=1/1 at 14:13:14Z), so the HelmRelease should clear on the next reconciliation.'
resource: flux-system/source-controller
alert_resource: crossplane-system/crossplane
tags:
    - runlore
    - incident
    - pod
    - flux-system
timestamp: "2026-11-09T14:15:49Z"
fingerprint: 3bfc8cef09ca34afda0817fd3f471b4336f3539145a81249b6462d940b5ac093
confidence: 0.85
origin: runlore
---

## Decision

- **why keep:** A Karpenter spot node was terminated (Server.SpotInstanceTermination) in the 'default' nodepool, evicting the source-controller pod. Its RWO EBS PVC (pvc-e4485222-724e-4456-8d08-81e8855476ad / vol-02e110f306702f13f, gp3 5Gi) was exclusively attached to the terminating node, so when the pod rescheduled to ip-10-0-38-1 at 14:07:07Z it hit 'FailedAttachVolume: Volume is already exclusively attached to one node, waiting on detach'. The source-controller pod stayed ContainerCreating/Pending for ~6 minutes (14:07:07Z→14:13:12Z), during which the source-controller.flux-system.svc.cluster.local Service (clusterIP 172.20.180.33) had no backing pods. helm-controller could not download the crossplane-2.4.3.tgz chart archive from source-controller, progressing through 'no such host' → 'i/o timeout' → 'context deadline exceeded' errors, producing the HelmRelease/crossplane ArtifactFailed. The source-controller has now recovered (EBS volume attached at 14:13:12Z, pod Running ready=1/1 at 14:13:14Z), so the HelmRelease should clear on the next reconciliation.
- **confidence:** 85%

## Symptom

HelmRelease/crossplane ArtifactFailed — Karpenter spot node termination evicted source-controller (RWO EBS PVC stuck on Multi-Attach), blocking chart download

Affected resource: Pod flux-system/source-controller-c6b548467-6qk5w

## Investigate

- cloud_resource_health: 'Karpenter nodepool default: instances=6 spot=6 terminated=1 spot-term-reasons=[Server.SpotInstanceTermination]'; EC2 i-012da89326f82f1c1 state=shutting-down
- kube_events (flux-system, source-controller-c6b548467-6qk5w): 14:07:07Z 'FailedAttachVolume: Waiting for detach for volume pvc-e4485222-724e-4456-8d08-81e8855476ad Volume is already exclusively attached to one node'; 14:13:12Z 'SuccessfulAttachVolume: AttachVolume.Attach succeeded for volume pvc-e4485222-724e-4456-8d08-81e8855476ad'
- controller_logs (helm-controller, crossplane): 14:06:36Z 'request failed: dial tcp: lookup source-controller.flux-system.svc.cluster.local. on 172.20.0.10:53: no such host'; 14:07:11Z 'dial tcp 172.20.180.33:80: i/o timeout'; 14:07:26Z 'context deadline exceeded' → 'Reconciler error: failed to download archive: GET http://source-controller.flux-system.svc.cluster.local./helmchart/crossplane-system/crossplane-system-crossplane/crossplane-2.4.3.tgz giving up after 3 attempt(s): context deadline exceeded'
- resource_spec (Pod source-controller-c6b548467-6qk5w): status.phase=Pending, containerStatuses[0].state.waiting.reason=ContainerCreating at time of inspection; pod_status now shows 'Running ready=1/1 age=7m' confirming recovery
- resource_spec (PV pvc-e4485222-724e-4456-8d08-81e8855476ad): accessModes=[ReadWriteOnce], csi.driver=ebs.csi.aws.com, nodeAffinity zone=eu-west-3c — RWO EBS cannot multi-attach during node termination
- query_metrics_range (up{pod=source-controller-...}): first=1 last=0, biggest jump -1@2026-11-09T14:06:00Z — source-controller went down at 14:06Z
- gitops_resource_status (HelmChart crossplane-system-crossplane): Ready=True (ChartPullSucceeded) 'pulled crossplane chart with version 2.4.3' at 14:04:25Z — chart was already cached, the failure is purely source-controller pod unavailability
- gitops_resource_status (HelmRepository crossplane-system/crossplane): Ready=True (Succeeded) — upstream repo is fine
- pod_status (crossplane-system): crossplane-98878f74d-rp4kp Running ready=2/2 age=4h — the Crossplane workload itself is healthy; this is a GitOps reconciliation failure, not a runtime outage
- kb_search: matching past incident '[Pre-existing/recurrent] karpenter-nodepools ArtifactFailed — EKS nodegroup rotation evicted source-controller' with identical RWO EBS PVC + Multi-Attach pattern, noting this has caused multiple ArtifactFailed incidents and recommending making source-controller resilient to node rotation
- kube_events (flux-system): 14:11:30Z 'Pod/helm-controller-5c65d49865-xjgjp FailedCreatePodSandBox: Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox ... plugin type="cilium-cni" failed (add): unable to allocate IP via local cilium agent: [POST /ipam][502] postIpamFailure "all CIDR ranges are exhausted"'
- kube_events (flux-system): 14:11:27Z 'Pod/helm-controller-5c65d49865-k7w28 Killing: Stopping container manager' — original helm-controller being evicted
- kb_search: matching runbook 'HelmRelease InstallFailed: a node at Cilium ENI IPAM capacity blocks a DaemonSet pod' documenting the same Cilium IPAM exhaustion pattern and resolution (free an IP on the exhausted node by deleting a replicated pod)

## Cause

1. **A Karpenter spot node was terminated (Server.SpotInstanceTermination) in the 'default' nodepool, evicting the source-controller pod. Its RWO EBS PVC (pvc-e4485222-724e-4456-8d08-81e8855476ad / vol-02e110f306702f13f, gp3 5Gi) was exclusively attached to the terminating node, so when the pod rescheduled to ip-10-0-38-1 at 14:07:07Z it hit 'FailedAttachVolume: Volume is already exclusively attached to one node, waiting on detach'. The source-controller pod stayed ContainerCreating/Pending for ~6 minutes (14:07:07Z→14:13:12Z), during which the source-controller.flux-system.svc.cluster.local Service (clusterIP 172.20.180.33) had no backing pods. helm-controller could not download the crossplane-2.4.3.tgz chart archive from source-controller, progressing through 'no such host' → 'i/o timeout' → 'context deadline exceeded' errors, producing the HelmRelease/crossplane ArtifactFailed. The source-controller has now recovered (EBS volume attached at 14:13:12Z, pod Running ready=1/1 at 14:13:14Z), so the HelmRelease should clear on the next reconciliation.** (85%)
2. **Secondary issue: node ip-10-0-43-30 has exhausted its Cilium per-node IPAM capacity ('all CIDR ranges are exhausted'), blocking pod sandbox creation for helm-controller's replacement pod (helm-controller-5c65d49865-xjgjp) at 14:11:30Z. The original helm-controller pod was being evicted ('Killing: Stopping container manager' at 14:11:27Z), and its replacement cannot get a pod IP on that node. This is the same Cilium ENI IPAM exhaustion pattern documented in a prior runbook. While the original helm-controller was still Running when checked, if it terminates before the IPAM issue resolves, Flux reconciliation across the cluster could be further impacted.** (35%)

## Resolution

- The immediate failure is self-healing — source-controller recovered at 14:13:14Z, so run 'flux -n crossplane-system reconcile helmrelease crossplane --force' to clear the ArtifactFailed state immediately. To prevent recurrence (this pattern has caused multiple incidents): (a) move source-controller to an on-demand Karpenter nodepool or EKS managed nodegroup exempt from spot interruption, OR (b) remove the RWO PVC dependency / switch to a storage class with faster detach-reattach, OR (c) add a PodDisruptionBudget and node anti-affinity so source-controller is not co-located on spot nodes being rotated. The KB runbook notes the RWO PVC + node affinity + spot nodes pattern has now caused at least 2 ArtifactFailed incidents. (reversible=true)
- Free a pod IP on node ip-10-0-43-30 by deleting a replicated pod scheduled there (a Deployment/ReplicaSet pod with replicas elsewhere — never the helm-controller pod itself). The freed IP allows helm-controller's replacement pod to start. If the node is persistently at capacity, consider spreading pods with topology spread constraints or using larger instance types for nodes that attract control-plane workloads. (reversible=true)

## Unresolved

- Whether the helm-controller original pod (helm-controller-5c65d49865-k7w28, which was being killed at 14:11:27Z) has fully terminated and whether its replacement (stuck on Cilium IPAM-exhausted node ip-10-0-43-30) has started. If helm-controller goes down, Flux reconciliation across the cluster will be further impacted beyond the crossplane HelmRelease.
- Whether the Karpenter spot termination was a routine spot reclamation or triggered by a broader capacity event. The cloud_what_changed output was truncated at 25 events, mostly SSM UpdateInstanceInformation noise, so the specific RunInstances/TerminateInstances for the terminated node could not be fully correlated.

