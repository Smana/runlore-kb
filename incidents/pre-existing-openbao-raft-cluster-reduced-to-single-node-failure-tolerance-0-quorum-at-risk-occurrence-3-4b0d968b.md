---
type: Incident
title: '[PRE-EXISTING] OpenBao raft cluster reduced to single node — failure tolerance 0, quorum at risk (occurrence #3,…'
description: 'OpenBao raft cluster has been reduced to a single peer (i-01a411d4725b2cce4) out of an originally multi-node cluster, leaving failure tolerance at 0. This is the SAME pre-existing condition identified in occurrence #2 (4h41m ago) — it has NOT been fixed. The cluster is functional (vault_autopilot_healthy=1, applied index steadily increasing 25918→38812 over 24h) but cannot tolerate any node loss: losing the sole remaining node means total quorum loss and inability to issue certificates or read secrets.'
resource: openbao/i-01a411d4725b2cce4
tags:
    - runlore
    - incident
    - ec2 instance
    - openbao
timestamp: "2026-10-07T12:04:35Z"
fingerprint: 4b0d968b4b130c4c082ad117c43a0c3a8e8d72c5a20eff3dc68e0d7841ea1d30
confidence: 0.82
---

## Decision

- **why keep:** OpenBao raft cluster has been reduced to a single peer (i-01a411d4725b2cce4) out of an originally multi-node cluster, leaving failure tolerance at 0. This is the SAME pre-existing condition identified in occurrence #2 (4h41m ago) — it has NOT been fixed. The cluster is functional (vault_autopilot_healthy=1, applied index steadily increasing 25918→38812 over 24h) but cannot tolerate any node loss: losing the sole remaining node means total quorum loss and inability to issue certificates or read secrets.
- **confidence:** 82%

## Symptom

[PRE-EXISTING] OpenBao raft cluster reduced to single node — failure tolerance 0, quorum at risk (occurrence #3, unfixed)

Affected resource: EC2 Instance openbao/i-01a411d4725b2cce4

## Investigate

- alert_rule OpenBaoRaftQuorumAtRisk: expr `vault_autopilot_failure_tolerance{job="openbao"} < 1` for 15m, state=firing
- query_metrics: vault_autopilot_failure_tolerance = 0 (current)
- query_metrics: vault_raft_peers = 1 — only one peer in the raft cluster
- query_metrics: vault_autopilot_node_healthy shows only node_id="i-01a411d4725b2cce4" = 1 — no other peers registered
- query_metrics_range (1440m): vault_raft_peers = 1 and vault_autopilot_failure_tolerance = 0 for the ENTIRE 24h lookback window — condition is long-standing, not a recent degradation
- query_metrics_range (1440m): up{job="openbao"} = 1 for the single instance bao.priv.aws.ogenki.io:8200 throughout — the surviving node has not crashed
- alert_rule OpenBaoRaftNodeLost (companion, also firing): expr `vault_autopilot_failure_tolerance < 2`, annotation states 'Usually a dead peer: instances replaced without `bao operator raft remove-peer` stay in the voter set, which spot reclamation does routinely'
- cloud_resource_health: EC2 i-01a411d4725b2cce4 state=running system=ok instance=ok — surviving node is healthy
- cloud_what_changed: i-01a411d4725b2cce4 was launched by AutoScaling (RunInstances by AutoScaling) at 2026-10-06T20:01:50Z and registered to a target group at 20:02:08Z
- list_resources Namespace: no 'openbao' namespace exists — OpenBao runs on EC2 instances outside Kubernetes, managed by an ASG, not GitOps
- Previous investigation (occurrence #2, 4h41m ago): concluded 'OpenBao raft cluster reduced to single node — failure tolerance 0, quorum at risk' — same condition, unfixed

## Cause

1. **OpenBao raft cluster has been reduced to a single peer (i-01a411d4725b2cce4) out of an originally multi-node cluster, leaving failure tolerance at 0. This is the SAME pre-existing condition identified in occurrence #2 (4h41m ago) — it has NOT been fixed. The cluster is functional (vault_autopilot_healthy=1, applied index steadily increasing 25918→38812 over 24h) but cannot tolerate any node loss: losing the sole remaining node means total quorum loss and inability to issue certificates or read secrets.** (82%)

## Resolution

- Two paths: (1) RESTORE QUORUM REDUNDANCY — bring up new OpenBao instances via the ASG and have them join the raft cluster (`bao operator raft add-peer`), then verify vault_raft_peers increases and failure_tolerance recovers to >=1 (ideally 2 for a 5-node cluster). (2) If dead peers are still in the voter set blocking cleanup, run `bao operator raft remove-peer <dead-peer-id>` on the surviving node to clean stale entries, then add fresh peers. The runbook at https://openbao.org/docs/concepts/integrated-storage/autopilot covers autopilot cleanup. Until peers are restored, this cluster is one instance termination away from total quorum loss. (reversible=true)

## Unresolved

- The exact ASG name managing OpenBao instances is unknown — cloud_resource_health reports EKS/Karpenter nodegroup status but not a separate OpenBao ASG. A human should identify the OpenBao ASG and check its desired/actual capacity to determine if new instances are being launched but failing to join the raft cluster, or if the ASG itself has been scaled down.
- Whether the dead peers are still in the raft voter set (requiring `bao operator raft remove-peer`) or have already been cleaned up (requiring only `add-peer` of new instances) — this determines which remediation path to take and can only be confirmed via `bao operator raft list-peers` on the surviving node.

