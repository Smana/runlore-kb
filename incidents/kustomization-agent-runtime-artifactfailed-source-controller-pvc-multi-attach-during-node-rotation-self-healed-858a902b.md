---
type: Incident
title: Kustomization/agent-runtime ArtifactFailed — source-controller PVC Multi-Attach during node rotation (self-healed,…
description: A Karpenter/EKS node rotation evicted the source-controller pod (source-controller-c6b548467-6qk5w) from its node. Its RWO EBS PVC (pvc-e4485222-724e-4456-8d08-81e8855476ad) was stuck in Multi-Attach — 'Volume is already exclusively attached to one node, waiting on detach before it can be attached to another node' (FailedAttachVolume at 14:07:07Z) — causing ~6 min of source-controller downtime (14:07→14:13). With source-controller unavailable, kustomize-controller could not download the OCIRepository artifact from the source-controller service (ClusterIP 172.20.180.33:80), resulting in ArtifactFailed for agent-runtime, agent-platform, and agents-nodepool at 14:12-14:13Z. The incident self-healed once the PVC detached and reattached at 14:13:12Z, with ReconciliationSucceeded by 14:18:00Z.
resource: flux-system/source-controller
alert_resource: flux-system/agent-runtime
tags:
    - runlore
    - incident
    - pod
    - flux-system
timestamp: "2026-11-09T15:10:09Z"
fingerprint: 858a902b711d9c609003bfe44630edd5768cda5f3a5a24d8b40692e43567705b
confidence: 0.85
origin: runlore
---

## Decision

- **why keep:** A Karpenter/EKS node rotation evicted the source-controller pod (source-controller-c6b548467-6qk5w) from its node. Its RWO EBS PVC (pvc-e4485222-724e-4456-8d08-81e8855476ad) was stuck in Multi-Attach — 'Volume is already exclusively attached to one node, waiting on detach before it can be attached to another node' (FailedAttachVolume at 14:07:07Z) — causing ~6 min of source-controller downtime (14:07→14:13). With source-controller unavailable, kustomize-controller could not download the OCIRepository artifact from the source-controller service (ClusterIP 172.20.180.33:80), resulting in ArtifactFailed for agent-runtime, agent-platform, and agents-nodepool at 14:12-14:13Z. The incident self-healed once the PVC detached and reattached at 14:13:12Z, with ReconciliationSucceeded by 14:18:00Z.
- **confidence:** 85%

## Symptom

Kustomization/agent-runtime ArtifactFailed — source-controller PVC Multi-Attach during node rotation (self-healed, recurrent)

Affected resource: Pod flux-system/source-controller-c6b548467-6qk5w

## Investigate

- kube_events(flux-system): 2026-11-09T14:07:07Z Warning Pod/source-controller-c6b548467-6qk5w FailedAttachVolume: 'Waiting for detach for volume pvc-e4485222-724e-4456-8d08-81e8855476ad Volume is already exclusively attached to one node, waiting on detach before it can be attached to another node'
- kube_events(flux-system): 2026-11-09T14:13:00Z Warning Kustomization/agent-runtime ArtifactFailed: 'failed to download archive: GET http://source-controller.flux-system.svc.cluster.local./ocirepository/flux-system/ai-native-ref/sha256:4a63df5581378742c635052d9d8152239bbe4fe7c37ccfcd3adf8ea7a6ccc3e9.tar.gz giving up after 10 attempt(s): dial tcp 172.20.180.33:80: i/o timeout'
- kube_events(flux-system): 2026-11-09T14:13:12Z Normal Pod/source-controller-c6b548467-6qk5w SuccessfulAttachVolume: 'AttachVolume.Attach succeeded for volume pvc-e4485222-724e-4456-8d08-81e8855476ad' followed by Container Created at 14:13:13 and Started at 14:13:14
- gitops_resource_status(Kustomization/agent-runtime): Ready=True (ReconciliationSucceeded) — 'Applied revision: sha256:4a63df5581378742c635052d9d8152239bbe4fe7c37ccfcd3adf8ea7a6ccc3e9'; ArtifactFailed event at 14:13:00Z, then ReconciliationSucceeded at 14:18:00Z
- pod_status(flux-system): source-controller-c6b548467-6qk5w Running ready=1/1 age=59m on node ip-10-0-38-1 — confirms pod was rescheduled ~59 min before inspection (~14:13Z), consistent with the recovery timeline
- kb_search: Pre-existing/recurrent incident 'karpenter-nodepools ArtifactFailed — EKS nodegroup rotation evicted source-controller (PVC Multi-Attach)' documents the exact same pattern: node rotation → source-controller PVC Multi-Attach → ~6 min downtime → ArtifactFailed across multiple Kustomizations. Noted as 2nd occurrence in ~4h at the time; this is now at least the 3rd.
- kube_events(flux-system): 2026-11-09T14:11:30Z Warning Pod/helm-controller-5c65d49865-xjgjp FailedCreatePodSandBox: 'cilium-cni failed (add): unable to allocate IP via local cilium agent: [POST /ipam][502] postIpamFailure all CIDR ranges are exhausted'
- kube_events(flux-system): 2026-11-09T14:12:28Z-14:16:28Z Warning Pod/helm-controller and Pod/source-watcher FailedCreatePodSandBox (x5): 'cilium-cni failed (add): unable to create endpoint: [PUT /endpoint/{id}][429] putEndpointIdTooManyRequests'
- kube_events(flux-system): 2026-11-09T14:06:51Z Warning FluxInstance/flux ArtifactFailed: 'fetch failed: fetching digest for artifact oci://ghcr.io/...failed: Get https://ghcr.io/v2/: dial tcp: lookup ghcr.io: i/o timeout'
- kube_events(flux-system): 2026-11-09T14:43:20Z Warning GitRepository/flux-system GitOperationFailed: 'lookup github.com on 172.20.0.10:53: read udp 100.64.179.112:55719->172.20.0.10:53: i/o timeout' — CoreDNS timeout
- query_metrics_range(cilium_operator_ipam_nodes): total nodes went 7→9→8 at 14:02-14:03Z, confirming node churn during the incident window; at-capacity=0 throughout (IPAM exhaustion was transient during pod rescheduling, not a standing capacity issue)

## Cause

1. **A Karpenter/EKS node rotation evicted the source-controller pod (source-controller-c6b548467-6qk5w) from its node. Its RWO EBS PVC (pvc-e4485222-724e-4456-8d08-81e8855476ad) was stuck in Multi-Attach — 'Volume is already exclusively attached to one node, waiting on detach before it can be attached to another node' (FailedAttachVolume at 14:07:07Z) — causing ~6 min of source-controller downtime (14:07→14:13). With source-controller unavailable, kustomize-controller could not download the OCIRepository artifact from the source-controller service (ClusterIP 172.20.180.33:80), resulting in ArtifactFailed for agent-runtime, agent-platform, and agents-nodepool at 14:12-14:13Z. The incident self-healed once the PVC detached and reattached at 14:13:12Z, with ReconciliationSucceeded by 14:18:00Z.** (85%)
2. **Compounding network disruption during the same node rotation: Cilium IPAM exhaustion ('all CIDR ranges are exhausted' at 14:11:30Z) and Cilium endpoint creation rate-limiting (putEndpointIdTooManyRequests 429 at 14:12-14:16Z) prevented helm-controller and source-watcher pods from getting network connectivity when rescheduling. Additionally, CoreDNS (172.20.0.10:53) experienced i/o timeouts causing DNS resolution failures for github.com, ghcr.io, and api.github.com from 14:06 through at least 14:43Z. These were secondary effects of the node churn, not the direct cause of the agent-runtime ArtifactFailed (which was the source-controller PVC Multi-Attach), but they broadened the disruption to multiple Flux components.** (35%)

## Resolution

- Make source-controller resilient to node rotation: (a) switch source-controller's PVC to a storage class that allows faster detach-reattach, or remove the node-affinity constraint; (b) schedule source-controller on Karpenter-provisioned nodes with interruption disruption budgets rather than nodes subject to rotation; or (c) pin source-controller to an on-demand node exempt from ASG/Karpenter rotation. This RWO PVC + node affinity + rotated nodes pattern has now caused 3+ ArtifactFailed incidents. No immediate action needed — the incident self-healed at 14:18Z. (reversible=true)

## Unresolved

- The specific trigger for the node rotation (EKS version update, Karpenter consolidation, spot interruption, or ASG refresh) could not be determined — CloudTrail results were dominated by SSM heartbeats and truncated before reaching the relevant EC2/Karpenter events around 14:02-14:07Z.

