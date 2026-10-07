---
type: Incident
title: '[PRE-EXISTING] OpenBao raft cluster reduced to single node — failure tolerance 0, quorum at risk (occurrence #2,…'
description: OpenBao raft cluster has been reduced to a single voter node (i-01a411d4725b2cce4), giving failure tolerance = 0. This is the SAME fault identified 9h53m ago — it has NOT been remediated. The cluster was originally multi-node (the OpenBaoRaftNodeLost alert annotation references a 'five-node cluster'), but spot instance reclamation evicted the other peers. The surviving node is functional (autopilot_healthy=1, commit index advancing), but losing it means total loss of quorum — no certificates can be issued and no secrets read.
resource: security/i-01a411d4725b2cce4
tags:
    - runlore
    - incident
    - ec2instance
    - security
timestamp: "2026-10-07T07:17:52Z"
fingerprint: 4064be1f3fcad1ac3de7f9ec28beeda5d9d291a376def44a0491f8b1371b5cda
confidence: 0.78
---

## Decision

- **why keep:** OpenBao raft cluster has been reduced to a single voter node (i-01a411d4725b2cce4), giving failure tolerance = 0. This is the SAME fault identified 9h53m ago — it has NOT been remediated. The cluster was originally multi-node (the OpenBaoRaftNodeLost alert annotation references a 'five-node cluster'), but spot instance reclamation evicted the other peers. The surviving node is functional (autopilot_healthy=1, commit index advancing), but losing it means total loss of quorum — no certificates can be issued and no secrets read.
- **confidence:** 78%

## Symptom

[PRE-EXISTING] OpenBao raft cluster reduced to single node — failure tolerance 0, quorum at risk (occurrence #2, unfixed)

Affected resource: EC2Instance security/i-01a411d4725b2cce4

## Investigate

- alert_rule OpenBaoRaftQuorumAtRisk: expr='vault_autopilot_failure_tolerance{job="openbao"} < 1' for=15m, state=firing
- query_metrics vault_autopilot_failure_tolerance = 0 (confirmed firing)
- query_metrics vault_autopilot_node_healthy shows only ONE node: node_id='i-01a411d4725b2cce4' = 1
- query_metrics vault_raft_storage_stats_applied_index and commit_index show only one peer_id='i-01a411d4725b2cce4' — no other peers in the raft cluster
- query_metrics_range vault_autopilot_failure_tolerance over 720m: first=0 last=0 min=0 — failure tolerance has been 0 since at least 2026-10-06T20:50:00Z, 12+ hours, well before incident start 2026-10-07T05:09:40Z
- cloud_resource_health EC2 i-01a411d4725b2cce4: state=running system=ok instance=ok — the surviving node is healthy
- query_metrics vault_autopilot_healthy = 1 and commit_index advancing (25918→34576) — the single node IS serving traffic
- alert_rule OpenBaoRaftNodeLost (also firing, threshold < 2) annotation: 'instances replaced without bao operator raft remove-peer stay in the voter set, which spot reclamation does routinely'
- cloud_what_changed shows BidEvictedEvent at 07:10:38Z — spot instance eviction in progress
- cloud_resource_health Karpenter: terminated=2 spot-term-reasons=[Server.SpotInstanceTermination] — spot terminations confirmed
- resource_spec Service security/openbao: type=ExternalName, externalName=bao.priv.aws.ogenki.io — OpenBao runs on EC2 outside Kubernetes, not as K8s pods
- pod_status namespace=security: no OpenBao pods exist (selector app.kubernetes.io/name=openbao matched no pods); list_resources StatefulSet in security: NONE — confirms OpenBao is not a K8s workload
- gitops_resource_status Kustomization flux-system/security-openbao: Ready=True (ReconciliationSucceeded); security-openbao-snapshot: Ready=True — GitOps layer is healthy, not the cause
- Previous investigation 9h53m ago concluded the same: 'OpenBao raft cluster reduced to single node — failure tolerance 0, quorum at risk' — this is occurrence #2, the fault was not fixed
- cloud_what_changed: BidEvictedEvent at 2026-10-07T07:10:38Z
- cloud_resource_health Karpenter: instances=8 spot=8 terminated=2 spot-term-reasons=[Server.SpotInstanceTermination]
- alert_rule OpenBaoRaftNodeLost annotation: 'Usually a dead peer: instances replaced without bao operator raft remove-peer stay in the voter set, which spot reclamation does routinely'

## Cause

1. **OpenBao raft cluster has been reduced to a single voter node (i-01a411d4725b2cce4), giving failure tolerance = 0. This is the SAME fault identified 9h53m ago — it has NOT been remediated. The cluster was originally multi-node (the OpenBaoRaftNodeLost alert annotation references a 'five-node cluster'), but spot instance reclamation evicted the other peers. The surviving node is functional (autopilot_healthy=1, commit index advancing), but losing it means total loss of quorum — no certificates can be issued and no secrets read.** (78%)
2. **Spot instance reclamation is the underlying mechanism that killed the peer nodes. CloudTrail shows BidEvictedEvent and Karpenter reports 2 spot terminations (Server.SpotInstanceTermination). The OpenBaoRaftNodeLost alert annotation explicitly identifies this pattern: spot reclamation removes instances without running 'bao operator raft remove-peer', leaving stale voters that eventually collapse the cluster.** (35%)

## Resolution

- Restore the OpenBao raft cluster to 3 or 5 nodes: (1) Launch new EC2 instances for OpenBao peers (avoid spot instances or use persistent EBS volumes with node IDs that survive replacement). (2) Join the new instances to the raft cluster with 'bao operator raft add-peer'. (3) Verify failure tolerance returns to >=1 (3-node) or >=2 (5-node). (4) Consider enabling OpenBao autopilot cleanup_dead_servers to automatically prune evicted spot peers. (5) If OpenBao peers run on spot instances, switch to on-demand or implement a lifecycle hook that runs 'bao operator raft remove-peer' before instance termination. Reference: https://openbao.org/docs/concepts/integrated-storage/autopilot (reversible=true)
- Move OpenBao peer instances off spot instances to on-demand, OR implement an ASG lifecycle hook / termination handler that runs 'bao operator raft remove-peer' before an instance is terminated. Enable autopilot cleanup_dead_servers=true so evicted peers are automatically pruned. (reversible=true)

## Unresolved

- Whether the OpenBao EC2 instances are managed by an ASG (auto-scaling group) or provisioned another way (Terraform/Crossplane) — this determines the remediation path for restoring peers
- Whether autopilot cleanup_dead_servers is enabled — if it were, dead peers should have been auto-pruned; the fact that only one node_id appears in metrics suggests either cleanup ran or peers were manually removed, but this could not be confirmed from available metrics
- The original cluster size (3 or 5 nodes) — the OpenBaoRaftNodeLost annotation references a 'five-node cluster' but this is not confirmed from live state

