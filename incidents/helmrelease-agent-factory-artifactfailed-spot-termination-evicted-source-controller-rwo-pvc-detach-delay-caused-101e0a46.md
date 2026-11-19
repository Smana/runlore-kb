---
type: Incident
title: HelmRelease/agent-factory ArtifactFailed — spot termination evicted source-controller; RWO PVC detach delay caused…
description: A Karpenter spot instance in the 'default' nodepool was reclaimed (Server.SpotInstanceTermination), evicting the source-controller pod (h5zfz) from its node at 09:45:24Z. The source-controller's chart-artifact EBS volume (PVC flux-system/source-controller, pvc-e33564c4, RWO gp3) could not detach from the terminating node, so the replacement pod (2psqx, scheduled 09:47:08Z) was stuck in FailedAttachVolume ('Volume is already exclusively attached to one node, waiting on detach') for ~6 minutes until 09:53:13Z. During this window source-controller was unavailable and its HTTP artifact endpoint was down, so helm-controller's attempt to fetch the agent-factory OCI chart artifact timed out ('context deadline exceeded') at 09:52:45Z, producing the HelmRelease ArtifactFailed. The release self-healed once the PVC attached at 09:53:13Z — source-controller came up at 09:53:17Z and HelmRelease went Ready=True (UpgradeSucceeded) at 09:53:29Z.
resource: flux-system/source-controller
alert_resource: agent-system/agent-factory
tags:
    - runlore
    - incident
    - pod
    - flux-system
timestamp: "2026-11-19T10:25:32Z"
fingerprint: 101e0a46936ab4620ad5896dad7ae29b926486e3794e9a12bdcac78d3e83f98a
confidence: 0.78
provenance:
    - Karpenter spot instance termination in 'default' nodepool at ~2026-11-19T09:42-09:45Z (Server.SpotInstanceTermination)
origin: runlore
---

## Decision

- **why keep:** A Karpenter spot instance in the 'default' nodepool was reclaimed (Server.SpotInstanceTermination), evicting the source-controller pod (h5zfz) from its node at 09:45:24Z. The source-controller's chart-artifact EBS volume (PVC flux-system/source-controller, pvc-e33564c4, RWO gp3) could not detach from the terminating node, so the replacement pod (2psqx, scheduled 09:47:08Z) was stuck in FailedAttachVolume ('Volume is already exclusively attached to one node, waiting on detach') for ~6 minutes until 09:53:13Z. During this window source-controller was unavailable and its HTTP artifact endpoint was down, so helm-controller's attempt to fetch the agent-factory OCI chart artifact timed out ('context deadline exceeded') at 09:52:45Z, producing the HelmRelease ArtifactFailed. The release self-healed once the PVC attached at 09:53:13Z — source-controller came up at 09:53:17Z and HelmRelease went Ready=True (UpgradeSucceeded) at 09:53:29Z.
- **confidence:** 78%
- **provenance:** Karpenter spot instance termination in 'default' nodepool at ~2026-11-19T09:42-09:45Z (Server.SpotInstanceTermination)

## Symptom

HelmRelease/agent-factory ArtifactFailed — spot termination evicted source-controller; RWO PVC detach delay caused ~6 min chart-fetch timeout (self-healed)

Affected resource: Pod flux-system/source-controller-c6b548467-2psqx

## Investigate

- cloud_resource_health: Karpenter nodepool default shows 'instances=6 spot=6 terminated=1 spot-term-reasons=[Server.SpotInstanceTermination]' — a spot instance was reclaimed
- kube_events (flux-system, pod source-controller-h5zfz): 09:45:24 'Killing: Stopping container manager'; 09:45:39 'ExceededGracePeriod: Container runtime did not kill the pod within specified grace period'; 09:45:25 'Nominated: Pod should schedule on: nodeclaim/default-x59ct, node/ip-10-0-13-15'
- kube_events (pod source-controller-2psqx): 09:47:08 'FailedAttachVolume: Waiting for detach for volume pvc-e33564c4-dd5b-4c65-be51-0e3a3bbc5b13 Volume is already exclusively attached to one node'; 09:53:13 'SuccessfulAttachVolume: AttachVolume.Attach succeeded for volume pvc-e33564c4-dd5b-4c65-be51-0e3a3bbc5b13'
- resource_spec PVC flux-system/source-controller: accessModes=[ReadWriteOnce], storageClassName=gp3, volumeName=pvc-e33564c4-dd5b-4c65-be51-0e3a3bbc5b13 — RWO EBS volume cannot multi-attach across nodes
- gitops_resource_status HelmRelease agent-system/agent-factory: Ready=True (UpgradeSucceeded), event at 09:52:45 'Warning ArtifactFailed(x4): failed to download archive: GET http://source-controller.flux-system.svc.cluster.local./ocirepository/agent-system/agent-factory/sha256:5c9930c4...tar.gz giving up after 2 attempt(s): context deadline exceeded'
- gitops_resource_status OCIRepository agent-system/agent-factory: Ready=True (Succeeded), artifact URL http://source-controller.flux-system.svc.cluster.local./ocirepository/agent-system/agent-factory/sha256:5c9930c4...tar.gz — the exact URL that timed out, confirming the failure was the source-controller HTTP endpoint being down, not the upstream OCI registry
- controller_logs source-controller (pod 2psqx) at 09:53:17Z: 'artifact up-to-date with remote revision sha256:5c9930c4...' — source-controller resumed serving immediately after PVC attach
- pod_status flux-system: source-controller-c6b548467-2psqx Running ready=1/1 age=34m (at investigation time) — pod recovered
- pod_status agent-system: agent-factory-67fd7766df-ltt7j and -zfgjb both Running ready=1/1 age=1h — workload fully operational
- kb_search matched a recurrent incident runbook: 'EKS nodegroup rotation evicted source-controller... RWO EBS PVC got stuck with Multi-Attach error, causing ~6 min of source-controller downtime... ArtifactFailed... Self-healed once the volume detached' — same RWO PVC + node churn pattern
- resource_spec PVC flux-system/source-controller: accessModes=[ReadWriteOnce], storageClassName=gp3 — RWO EBS volumes are node-affinity-bound and cannot multi-attach, forcing a detach-then-attach serial delay during pod rescheduling
- kube_events: FailedAttachVolume at 09:47:08Z → SuccessfulAttachVolume at 09:53:13Z = ~6 min gap, during which ArtifactFailed occurred at 09:52:45Z
- kb_search runbook (incidents/pre-existing-recurrent-karpenter-nodepools-artifactfailed...): 'Its RWO EBS PVC is node-affinity-bound and got stuck with a Multi-Attach error... causing ~6 min of source-controller downtime... has now caused 2 ArtifactFailed incidents in ~4h' — this is a known recurrent pattern, not a one-off

## Cause

1. **A Karpenter spot instance in the 'default' nodepool was reclaimed (Server.SpotInstanceTermination), evicting the source-controller pod (h5zfz) from its node at 09:45:24Z. The source-controller's chart-artifact EBS volume (PVC flux-system/source-controller, pvc-e33564c4, RWO gp3) could not detach from the terminating node, so the replacement pod (2psqx, scheduled 09:47:08Z) was stuck in FailedAttachVolume ('Volume is already exclusively attached to one node, waiting on detach') for ~6 minutes until 09:53:13Z. During this window source-controller was unavailable and its HTTP artifact endpoint was down, so helm-controller's attempt to fetch the agent-factory OCI chart artifact timed out ('context deadline exceeded') at 09:52:45Z, producing the HelmRelease ArtifactFailed. The release self-healed once the PVC attached at 09:53:13Z — source-controller came up at 09:53:17Z and HelmRelease went Ready=True (UpgradeSucceeded) at 09:53:29Z.** (78%) — change: Karpenter spot instance termination in 'default' nodepool at ~2026-11-19T09:42-09:45Z (Server.SpotInstanceTermination)
2. **The source-controller's use of a ReadWriteOnce (RWO) gp3 EBS PVC for artifact storage makes it inherently fragile during node rotation: the EBS volume must fully detach from the old node before it can attach to the new one, creating an unavoidable multi-minute gap during which source-controller cannot serve artifacts. This is a recurrent design weakness — the same pattern has caused prior ArtifactFailed incidents per the knowledge base, and will recur on every spot reclamation or node rotation that lands on the source-controller's node.** (50%)

## Resolution

- No action suggested by the investigation — see `## Unresolved`.

## Unresolved

- The specific Karpenter NodeClaim/EC2 instance ID that was spot-terminated could not be pinpointed — cloud_resource_health reports 'terminated=1' for the default nodepool but does not name the instance, and the old source-controller pod's node was not captured in pod_status (pod h5zfz is already deleted). The node ip-10-0-11-129 went NotReady at 09:44 per kube_node_status_condition metrics, which is likely the terminated node, but this is inferred from the timing correlation rather than directly confirmed.
- Whether the Karpenter default nodepool should be configured to use on-demand instances for critical control-plane workloads (source-controller, kustomize-controller, etc.) is a capacity/cost tradeoff that requires human judgment about the cluster's spot interruption tolerance.

## Citations

[1] Karpenter spot instance termination in 'default' nodepool at ~2026-11-19T09:42-09:45Z (Server.SpotInstanceTermination)

