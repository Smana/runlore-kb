---
type: Incident
title: 'OpenBaoRaftQuorumAtRisk — false positive: FT=0 is the mathematical norm for this single-node dev OpenBao raft cluster'
description: 'The OpenBao instance in this dev environment is a single-node raft cluster. For a 1-node raft cluster, failure tolerance = floor((1-1)/2) = 0 by definition. The alert rule vault_autopilot_failure_tolerance < 1 (and the companion OpenBaoRaftNodeLost at < 2) are configured with thresholds appropriate for a multi-node HA cluster and are inappropriate for this single-node dev deployment, making this alert a false positive. The cluster is fully healthy: autopilot reports healthy, the single node is healthy and unsealed, raft is actively committing entries, and the EC2 instance passes status checks.'
resource: openbao/bao.priv.aws.ogenki.io:8200
tags:
    - runlore
    - incident
    - openbao
timestamp: "2026-09-16T08:10:52Z"
fingerprint: ff89cc55d6197fbd6aa422d76b1400e3bc46a2ccb5afefea508f169c3b423e20
confidence: 0.9
---

## Decision

- **why keep:** The OpenBao instance in this dev environment is a single-node raft cluster. For a 1-node raft cluster, failure tolerance = floor((1-1)/2) = 0 by definition. The alert rule vault_autopilot_failure_tolerance < 1 (and the companion OpenBaoRaftNodeLost at < 2) are configured with thresholds appropriate for a multi-node HA cluster and are inappropriate for this single-node dev deployment, making this alert a false positive. The cluster is fully healthy: autopilot reports healthy, the single node is healthy and unsealed, raft is actively committing entries, and the EC2 instance passes status checks.
- **confidence:** 90%

## Symptom

OpenBaoRaftQuorumAtRisk — false positive: FT=0 is the mathematical norm for this single-node dev OpenBao raft cluster

Affected resource: OpenBao openbao/bao.priv.aws.ogenki.io:8200

## Investigate

- alert_rule: OpenBaoRaftQuorumAtRisk expr = vault_autopilot_failure_tolerance{job="openbao"} < 1, for=15m, state=firing. The metric value is 0, confirming the alert fires.
- query_metrics: vault_autopilot_node_healthy{node_id="i-0a8ae89813d5970f9"} = 1 — exactly ONE node in the cluster
- query_metrics: vault_raft_storage_stats_applied_index{peer_id="i-0a8ae89813d5970f9"} = 5231, commit_index = 5231 — only ONE peer_id exists, no stale voters
- query_metrics: count(vault_autopilot_node_healthy{job="openbao"}) = 1 — steady at 1 node for the entire 24h lookback (query_metrics_range 1440m)
- query_metrics: vault_autopilot_healthy = 1 — autopilot considers the cluster healthy
- query_metrics: vault_core_unsealed{exported_cluster="vault-cluster-f88e2bb4"} = 1 — the node is unsealed and operational
- query_metrics_range: vault_autopilot_failure_tolerance = 0 flat for 24h (since_minutes=1440) — this is steady state, not a new degradation from a multi-node cluster losing peers
- query_metrics_range: vault_raft_storage_stats_applied_index steadily climbing 4899→5250 over 120m — raft is actively committing log entries
- cloud_resource_health: EC2 i-0a8ae89813d5970f9 state=running, system=ok, instance=ok
- alert_rule OpenBaoRaftNodeLost annotation: 'Failure tolerance has been below 2 for 15 minutes, so a five-node cluster is no longer tolerating the two failures it is sized for' — this 5-node assumption is wrong for this single-node dev cluster

## Cause

1. **The OpenBao instance in this dev environment is a single-node raft cluster. For a 1-node raft cluster, failure tolerance = floor((1-1)/2) = 0 by definition. The alert rule vault_autopilot_failure_tolerance < 1 (and the companion OpenBaoRaftNodeLost at < 2) are configured with thresholds appropriate for a multi-node HA cluster and are inappropriate for this single-node dev deployment, making this alert a false positive. The cluster is fully healthy: autopilot reports healthy, the single node is healthy and unsealed, raft is actively committing entries, and the EC2 instance passes status checks.** (90%)

## Resolution

- Suppress or reconfigure the OpenBaoRaftQuorumAtRisk (FT < 1) and OpenBaoRaftNodeLost (FT < 2) alert rules for the dev environment / single-node OpenBao deployments. Options: (1) disable these alerts for env=dev, (2) add a condition like 'and count(vault_autopilot_node_healthy) > 1' so they only fire when a multi-node cluster actually loses redundancy, or (3) set the threshold to FT < 0 (impossible value) to effectively disable for single-node. No emergency action is needed — the cluster is healthy and operational. (reversible=true)

## Unresolved

- Why did the OpenBao process restart at ~07:37 UTC? The EC2 instance is healthy and running, but the restart cause (deploy, spot reclamation+relaunch, manual restart) could not be determined — OpenBao is external and its logs/systemd journal are not accessible from the available tools. This is a minor point: the cluster recovered cleanly and FT was 0 before and after.

