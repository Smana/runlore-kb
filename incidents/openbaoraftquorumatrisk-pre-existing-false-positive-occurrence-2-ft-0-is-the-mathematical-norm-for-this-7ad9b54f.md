---
type: Incident
title: 'OpenBaoRaftQuorumAtRisk — PRE-EXISTING false positive (occurrence #2): FT=0 is the mathematical norm for this…'
description: The alert rule vault_autopilot_failure_tolerance{job="openbao"} < 1 fires as a false positive because this is a single-node dev OpenBao raft cluster where FT=0 is the mathematical norm, not a symptom of a dead peer. For a raft cluster of N=1, failure tolerance = floor((N-1)/2) = 0. FT=0 means the cluster cannot tolerate a node failure — it does NOT mean quorum is lost. The cluster is healthy, unsealed, active, and serving traffic. The companion OpenBaoRaftNodeLost alert (threshold FT < 2) is also a false positive for the same reason. This is the same condition identified 3h37m ago; it has persisted unchanged.
resource: openbao/openbao
tags:
    - runlore
    - incident
    - service
    - openbao
timestamp: "2026-09-16T11:52:29Z"
fingerprint: 7ad9b54fdd065acc50cfea643e5d08831c7ad73331c8d55c1146fb70176a65e4
confidence: 0.88
provenance:
    - none — FT has been flat at 0 for the entire 240-minute observation window; no GitOps change, no node death, no peer loss
---

## Decision

- **why keep:** The alert rule vault_autopilot_failure_tolerance{job="openbao"} < 1 fires as a false positive because this is a single-node dev OpenBao raft cluster where FT=0 is the mathematical norm, not a symptom of a dead peer. For a raft cluster of N=1, failure tolerance = floor((N-1)/2) = 0. FT=0 means the cluster cannot tolerate a node failure — it does NOT mean quorum is lost. The cluster is healthy, unsealed, active, and serving traffic. The companion OpenBaoRaftNodeLost alert (threshold FT < 2) is also a false positive for the same reason. This is the same condition identified 3h37m ago; it has persisted unchanged.
- **confidence:** 88%
- **provenance:** none — FT has been flat at 0 for the entire 240-minute observation window; no GitOps change, no node death, no peer loss

## Symptom

OpenBaoRaftQuorumAtRisk — PRE-EXISTING false positive (occurrence #2): FT=0 is the mathematical norm for this single-node dev OpenBao raft cluster

Affected resource: Service openbao/openbao

## Investigate

- alert_rule: OpenBaoRaftQuorumAtRisk expr = vault_autopilot_failure_tolerance{job="openbao"} < 1, state=firing; OpenBaoRaftNodeLost expr = vault_autopilot_failure_tolerance{job="openbao"} < 2, state=firing — both thresholds assume a multi-node cluster
- query_metrics: vault_autopilot_failure_tolerance{job="openbao"} = 0 (the alert's exact metric)
- query_metrics_range: vault_autopilot_failure_tolerance has been flat at 0 for the entire 240-min window (first=0 last=0 min=0 max=0) — this is steady state, not a degradation
- query_metrics: vault_autopilot_node_healthy shows exactly ONE node_id='i-0a8ae89813d5970f9' = 1 — only one raft peer exists
- query_metrics: vault_raft_storage_stats_applied_index has exactly one peer_id='i-0a8ae89813d5970f9' — single-peer confirmation from raft storage layer
- query_metrics_range: count of distinct peer_ids = 1 for entire 240-min window — the cluster has always been single-node
- query_metrics: vault_autopilot_healthy=1, vault_core_unsealed=1, vault_core_active=1, vault_core_in_flight_requests=0 — OpenBao is fully operational
- cloud_resource_health: EC2 i-0a8ae89813d5970f9 state=running system=ok instance=ok — the one node is healthy, there is no dead peer
- query_metrics_range: process_start_time_seconds flat across 240-min window — no restart during the incident; the process was already running before the window started

## Cause

1. **The alert rule vault_autopilot_failure_tolerance{job="openbao"} < 1 fires as a false positive because this is a single-node dev OpenBao raft cluster where FT=0 is the mathematical norm, not a symptom of a dead peer. For a raft cluster of N=1, failure tolerance = floor((N-1)/2) = 0. FT=0 means the cluster cannot tolerate a node failure — it does NOT mean quorum is lost. The cluster is healthy, unsealed, active, and serving traffic. The companion OpenBaoRaftNodeLost alert (threshold FT < 2) is also a false positive for the same reason. This is the same condition identified 3h37m ago; it has persisted unchanged.** (88%) — change: none — FT has been flat at 0 for the entire 240-minute observation window; no GitOps change, no node death, no peer loss

## Resolution

- Fix the alerting rule to account for single-node dev clusters. Options: (1) add an exclusion for env=dev or clusters with num_peers=1; (2) change the threshold to be relative to cluster size, e.g. vault_autopilot_failure_tolerance < (vault_raft_num_peers - 1) / 2; (3) add a condition like vault_raft_num_peers > 1 to the alert expression so it only fires when a multi-node cluster actually loses redundancy. No action is needed on the OpenBao instance itself — it is healthy and serving. (reversible=true)

## Citations

[1] none — FT has been flat at 0 for the entire 240-minute observation window; no GitOps change, no node death, no peer loss

