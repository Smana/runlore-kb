---
type: Incident
title: Kustomization/zitadel ReconciliationFailed — cert-manager-webhook had no endpoints during spot node termination…
description: 'Spot node terminations triggered a cluster-wide capacity shortage: 3 of 8 Karpenter default-nodepool spot instances were terminated (Server.SpotInstanceTermination), reducing schedulable capacity so that critical control-plane pods (cert-manager-webhook, cert-manager, kyverno-cleanup-controller, cloudnative-pg operator) hit FailedScheduling (''0/7 nodes available: 1 Insufficient memory, 2-3 Insufficient cpu, 4 node(s) had untolerated taints''). With the cert-manager-webhook pod unable to schedule, the cert-manager-webhook Kubernetes Service had no endpoints, so the Certificate/security/zitadel dry-run validation call failed (''no endpoints available for service cert-manager-webhook''), causing the Flux Kustomization/zitadel to report ReconciliationFailed. The same capacity shortage also knocked out the CNPG webhook (cnpg-webhook-service) and led to eviction of the CNPG cluster''s primary pod, triggering a database failover that left zitadel unable to reach its PostgreSQL backend.'
resource: flux-system/zitadel
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-09-26T23:55:18Z"
fingerprint: a63d8db4c48e8f5421057f9c0f66a58a43f0a70dd390587ac2ef215f7f02dba7
confidence: 0.75
provenance:
    - Karpenter spot termination event (Server.SpotInstanceTermination) at ~23:46Z
    - Spot eviction of CNPG cluster pods at ~23:46Z triggering failover and volume reattachment
---

## Decision

- **why keep:** Spot node terminations triggered a cluster-wide capacity shortage: 3 of 8 Karpenter default-nodepool spot instances were terminated (Server.SpotInstanceTermination), reducing schedulable capacity so that critical control-plane pods (cert-manager-webhook, cert-manager, kyverno-cleanup-controller, cloudnative-pg operator) hit FailedScheduling ('0/7 nodes available: 1 Insufficient memory, 2-3 Insufficient cpu, 4 node(s) had untolerated taints'). With the cert-manager-webhook pod unable to schedule, the cert-manager-webhook Kubernetes Service had no endpoints, so the Certificate/security/zitadel dry-run validation call failed ('no endpoints available for service cert-manager-webhook'), causing the Flux Kustomization/zitadel to report ReconciliationFailed. The same capacity shortage also knocked out the CNPG webhook (cnpg-webhook-service) and led to eviction of the CNPG cluster's primary pod, triggering a database failover that left zitadel unable to reach its PostgreSQL backend.
- **confidence:** 75%
- **provenance:** Karpenter spot termination event (Server.SpotInstanceTermination) at ~23:46Z, Spot eviction of CNPG cluster pods at ~23:46Z triggering failover and volume reattachment

## Symptom

Kustomization/zitadel ReconciliationFailed — cert-manager-webhook had no endpoints during spot node termination cascade

Affected resource: Kustomization flux-system/zitadel

## Investigate

- cloud_resource_health: 'Karpenter nodepool default: instances=8 spot=8 terminated=3 spot-term-reasons=[Server.SpotInstanceTermination]'
- kube_events (security): 'Pod/cert-manager-webhook-7bc94bf9cd-h9sd6 FailedScheduling (x25): 0/7 nodes are available: 1 Insufficient memory, 2 Insufficient cpu, 4 node(s) had untolerated taints' at 23:46:10Z
- kube_events (security): 'Pod/cert-manager-webhook-7bc94bf9cd-h9sd6 FailedCreatePodSandBox: ... cilium-cni failed (add): unable to create endpoint: [PUT /endpoint/{id}][429] putEndpointIdTooManyRequests' at 23:47:31Z — compounding Cilium IP exhaustion
- kube_events (security): 'Pod/cert-manager-webhook-7bc94bf9cd-h9sd6 FailedCreatePodSandBox: ... unable to allocate IP via local cilium agent: [POST /ipam][502] postIpamFailure "all CIDR ranges are exhausted"' at 23:46:56Z
- gitops_resource_status (Kustomization/zitadel): Events show 'ReconciliationFailed(x2): Certificate/security/zitadel dry-run failed (InternalError): failed calling webhook webhook.cert-manager.io: no endpoints available for service cert-manager-webhook' at 23:47:45Z, followed by 'ReconciliationSucceeded' at 23:50Z
- kube_events (security): 'SQLInstance/xplane-zitadel ComposeResources: ... failed calling webhook mscheduledbackup.cnpg.io: no endpoints available for service cnpg-webhook-service' at 23:47:32Z — same pattern affecting CNPG webhook
- kube_events (security): 'Cluster/xplane-zitadel-cnpg-cluster FailingOver: Current primary isn't healthy, initiating a failover from xplane-zitadel-cnpg-cluster-2' at 23:52:13Z
- pod_logs (zitadel): 'failed to connect to user=zitadel database=zitadel: 172.20.154.254:5432 (xplane-zitadel-cnpg-cluster-rw): dial error: dial tcp 172.20.154.254:5432: connect: connection refused' (x36+, 23:50:10Z→23:50:47Z)
- what_changed (flux-system/zitadel): only a sync at 23:50:19Z with no Git diff — no application change caused this; it is purely an infrastructure capacity event
- resource_spec (Cluster/xplane-zitadel-cnpg-cluster): status.conditions[Ready]=False, reason=ClusterIsNotReady; phase='Waiting for the instances to become active'; instancesStatus: failed=[cluster-2], healthy=[cluster-1]; readyInstances=1
- pod_status: xplane-zitadel-cnpg-cluster-2 is Pending ready=0/3 on node ip-10-0-27-133
- kube_events (cluster-2): 'FailedAttachVolume: Waiting for detach for volume pvc-59b01a7f-5494-46b1-8bad-7c1e575065ef Volume is already exclusively attached to one node, waiting on detach' at 23:52:33Z
- kube_events (cluster-1): 'FailedAttachVolume (x4): CSINode ip-10-0-28-118 does not contain driver ebs.csi.aws.com' at 23:47:37Z → then 'SuccessfulAttachVolume' at 23:47:42Z (recovered)
- cloud_what_changed: repeated 'AttachVolume ... FAILED: Client.VolumeInUse (vol-08bbd6c5f8d5631a2 is already attached to an instance)' at 23:50:24Z→23:50:50Z — EBS CSI struggling with cross-node volume reattachment

## Cause

1. **Spot node terminations triggered a cluster-wide capacity shortage: 3 of 8 Karpenter default-nodepool spot instances were terminated (Server.SpotInstanceTermination), reducing schedulable capacity so that critical control-plane pods (cert-manager-webhook, cert-manager, kyverno-cleanup-controller, cloudnative-pg operator) hit FailedScheduling ('0/7 nodes available: 1 Insufficient memory, 2-3 Insufficient cpu, 4 node(s) had untolerated taints'). With the cert-manager-webhook pod unable to schedule, the cert-manager-webhook Kubernetes Service had no endpoints, so the Certificate/security/zitadel dry-run validation call failed ('no endpoints available for service cert-manager-webhook'), causing the Flux Kustomization/zitadel to report ReconciliationFailed. The same capacity shortage also knocked out the CNPG webhook (cnpg-webhook-service) and led to eviction of the CNPG cluster's primary pod, triggering a database failover that left zitadel unable to reach its PostgreSQL backend.** (55%) — change: Karpenter spot termination event (Server.SpotInstanceTermination) at ~23:46Z
2. **The CNPG PostgreSQL cluster for zitadel is only partially recovered: cluster-1 is Running as the new primary (promoted at 23:52Z), but cluster-2 is Pending because its EBS volume is exclusively attached to another node ('Volume is already exclusively attached to one node, waiting on detach'). The cluster reports Ready=False with phase 'Waiting for the instances to become active' and only 1/2 healthy instances. This is a secondary failure caused by the same spot node evictions — both CNPG pods were evicted, and the failover + volume reattachment cycle is still in progress.** (75%) — change: Spot eviction of CNPG cluster pods at ~23:46Z triggering failover and volume reattachment

## Resolution

- No GitOps change to revert — this is an infrastructure capacity event. The Kustomization has already self-healed (Ready=True at 23:50Z) and zitadel pod is Ready=1/1. To prevent recurrence, review Karpenter spot disruption budgets / consolidation policies to reduce simultaneous spot terminations, and consider adding anti-affinity or priority class for cert-manager-webhook so it schedules before workloads. Also address Cilium IP exhaustion by enlarging pod CIDR ranges. (reversible=true)
- Monitor cluster-2 volume detachment/reattachment — it should resolve automatically once the EBS CSI driver detaches the volume from the old (terminated) node. If cluster-2 stays Pending beyond ~15 minutes, manually force-detach the stale volume attachment in AWS or delete the stuck pod to let the operator recreate it. Confirm the CNPG cluster reaches Ready=True with 2/2 instances. (reversible=true)

## Unresolved

- Whether Cilium IP exhaustion ('all CIDR ranges are exhausted') is a persistent capacity issue or was transiently caused by the burst of pod restarts during spot node churn — this determines whether enlarging the pod CIDR is needed as a preventive fix.
- Whether cluster-2 will self-heal or requires manual volume force-detach intervention — needs monitoring over the next ~10-15 minutes.

## Citations

[1] Karpenter spot termination event (Server.SpotInstanceTermination) at ~23:46Z
[2] Spot eviction of CNPG cluster pods at ~23:46Z triggering failover and volume reattachment

