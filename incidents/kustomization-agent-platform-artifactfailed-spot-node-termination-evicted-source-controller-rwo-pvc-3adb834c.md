---
type: Incident
title: Kustomization/agent-platform ArtifactFailed — spot node termination evicted source-controller (RWO PVC…
description: 'Karpenter spot node terminations (cloud_resource_health: nodepool default terminated=1 with spot-term-reasons=[Server.SpotInstanceTermination], nodepool io terminated=1) at ~14:02-14:05Z evicted the source-controller pod (source-controller-c6b548467-nrcld, force-killed via ExceededGracePeriod at 14:05:31Z). Its replacement pod (source-controller-c6b548467-6qk5w) could not start for ~6 minutes because its RWO EBS PVC (pvc-e4485222-724e-4456-8d08-81e8855476ad) was stuck in a Multi-Attach state — kube_events at 14:07:07Z: ''Waiting for detach for volume pvc-e4485222... Volume is already exclusively attached to one node, waiting on detach before it can be attached to another node''. The pod was Pending from 14:07:30Z to 14:13:30Z (metrics confirm). With no healthy source-controller pod serving the ClusterIP 172.20.180.33:80, Kustomization/agent-platform (and agent-runtime, agents-nodepool) could not download the OCIRepository ai-native-ref artifact and reported ArtifactFailed ''dial tcp 172.20.180.33:80: i/o timeout'' at 14:12:40Z. The Kustomization self-healed at 14:15:51Z once the volume detached and the pod started serving. This is the SAME root cause pattern documented in the pre-existing recurrent KB incident (source-controller RWO PVC + node rotation → Multi-Attach → ArtifactFailed), now recurring on 2026-11-09.'
resource: flux-system/source-controller
alert_resource: flux-system/agent-platform
tags:
    - runlore
    - incident
    - pod
    - flux-system
timestamp: "2026-11-09T15:06:29Z"
fingerprint: 3adb834c71e9a2e35055cb96aa23ca80b399f726041dd218832474fefef1c4a5
confidence: 0.82
origin: runlore
---

## Decision

- **why keep:** Karpenter spot node terminations (cloud_resource_health: nodepool default terminated=1 with spot-term-reasons=[Server.SpotInstanceTermination], nodepool io terminated=1) at ~14:02-14:05Z evicted the source-controller pod (source-controller-c6b548467-nrcld, force-killed via ExceededGracePeriod at 14:05:31Z). Its replacement pod (source-controller-c6b548467-6qk5w) could not start for ~6 minutes because its RWO EBS PVC (pvc-e4485222-724e-4456-8d08-81e8855476ad) was stuck in a Multi-Attach state — kube_events at 14:07:07Z: 'Waiting for detach for volume pvc-e4485222... Volume is already exclusively attached to one node, waiting on detach before it can be attached to another node'. The pod was Pending from 14:07:30Z to 14:13:30Z (metrics confirm). With no healthy source-controller pod serving the ClusterIP 172.20.180.33:80, Kustomization/agent-platform (and agent-runtime, agents-nodepool) could not download the OCIRepository ai-native-ref artifact and reported ArtifactFailed 'dial tcp 172.20.180.33:80: i/o timeout' at 14:12:40Z. The Kustomization self-healed at 14:15:51Z once the volume detached and the pod started serving. This is the SAME root cause pattern documented in the pre-existing recurrent KB incident (source-controller RWO PVC + node rotation → Multi-Attach → ArtifactFailed), now recurring on 2026-11-09.
- **confidence:** 82%

## Symptom

Kustomization/agent-platform ArtifactFailed — spot node termination evicted source-controller (RWO PVC Multi-Attach delay), self-healed in ~3 min

Affected resource: Pod flux-system/source-controller-c6b548467-6qk5w

## Investigate

- gitops_resource_status Kustomization/agent-platform: Ready=True (ReconciliationSucceeded) with Warning ArtifactFailed event at 2026-11-09T14:12:40Z matching the incident message exactly, followed by Normal ReconciliationSucceeded at 14:15:51Z (~3 min later)
- kube_events flux-system: '14:07:07Z Warning Pod/source-controller-c6b548467-6qk5w FailedAttachVolume: Waiting for detach for volume pvc-e4485222-724e-4456-8d08-81e8855476ad Volume is already exclusively attached to one node, waiting on detach before it can be attached to another node'
- kube_events: '14:05:31Z Warning Pod/source-controller-c6b548467-nrcld ExceededGracePeriod: Container runtime did not kill the pod within specified grace period.' — the prior source-controller pod was force-killed on a terminating node
- query_metrics_range kube_pod_status_phase source-controller-c6b548467-6qk5w phase=Pending: first=1 last=0 min=0@14:13:30Z max=1@14:07:30Z — pod was Pending (unschedulable) from 14:07:30 to 14:13:30, exactly bracketing the ArtifactFailed window
- query_metrics_range node Ready condition: ip-10-0-38-1 NotReady max=1@14:02Z recovered 14:03Z; ip-10-0-40-151 NotReady max=1@14:04Z (still down at end); ip-10-0-43-30 NotReady 14:10-14:11; ip-10-0-43-97 NotReady from 14:13Z — multiple simultaneous node failures consistent with spot termination
- cloud_resource_health: 'Karpenter nodepool default: instances=6 spot=6 terminated=1 spot-term-reasons=[Server.SpotInstanceTermination]; Karpenter nodepool io: instances=2 spot=2 terminated=1' — confirms spot instance terminations
- kb_search returned pre-existing recurrent incident: 'EKS managed-nodegroup node rotation evicted the source-controller pod. Its RWO EBS PVC is node-affinity-bound and got stuck with a Multi-Attach error, causing ~6 min of source-controller downtime. With source-controller unavailable, the ExternalArtifact/infra-artifact source could not be served, and karpenter-nodepools reported ArtifactFailed.' — identical mechanism, same resolution recommended
- incident_timeline flux-system: 14:05:31 ExceededGracePeriod → 14:07:07 FailedAttachVolume (Multi-Attach) → 14:12:40 ArtifactFailed Kustomization/agent-platform → 14:15:51 ReconciliationSucceeded (from gitops_resource_status events)

## Cause

1. **Karpenter spot node terminations (cloud_resource_health: nodepool default terminated=1 with spot-term-reasons=[Server.SpotInstanceTermination], nodepool io terminated=1) at ~14:02-14:05Z evicted the source-controller pod (source-controller-c6b548467-nrcld, force-killed via ExceededGracePeriod at 14:05:31Z). Its replacement pod (source-controller-c6b548467-6qk5w) could not start for ~6 minutes because its RWO EBS PVC (pvc-e4485222-724e-4456-8d08-81e8855476ad) was stuck in a Multi-Attach state — kube_events at 14:07:07Z: 'Waiting for detach for volume pvc-e4485222... Volume is already exclusively attached to one node, waiting on detach before it can be attached to another node'. The pod was Pending from 14:07:30Z to 14:13:30Z (metrics confirm). With no healthy source-controller pod serving the ClusterIP 172.20.180.33:80, Kustomization/agent-platform (and agent-runtime, agents-nodepool) could not download the OCIRepository ai-native-ref artifact and reported ArtifactFailed 'dial tcp 172.20.180.33:80: i/o timeout' at 14:12:40Z. The Kustomization self-healed at 14:15:51Z once the volume detached and the pod started serving. This is the SAME root cause pattern documented in the pre-existing recurrent KB incident (source-controller RWO PVC + node rotation → Multi-Attach → ArtifactFailed), now recurring on 2026-11-09.** (82%)

## Resolution

- Make source-controller resilient to spot node termination, as recommended in the matching KB runbook: (a) schedule source-controller on on-demand nodes exempt from spot rotation rather than Karpenter spot nodepools, or (b) switch its PVC storage class to one allowing faster detach-reattach / use EFS (RWX) instead of RWO EBS, or (c) add a PodDisruptionBudget and node affinity to pin it away from spot/rotated nodes. This pattern has now caused 3+ ArtifactFailed incidents (2 on 2026-09-08 per KB, 1 on 2026-11-09) and will keep recurring on every spot reclamation that hits the source-controller node. (reversible=true)

## Unresolved

- The specific trigger for the simultaneous spot node terminations at ~14:02Z could not be pinpointed — cloud_what_changed showed only SSM UpdateInstanceInformation and Karpenter CreateLaunchTemplate/DeleteLaunchTemplate/DryRun RunInstances events from 14:56Z onward, not the actual TerminateInstances calls around 14:02-14:05Z (they may be outside the 75-minute CloudTrail lookback or truncated by the 25-event cap). cloud_resource_health confirms Server.SpotInstanceTermination as the reason but not the initiating AWS event.

