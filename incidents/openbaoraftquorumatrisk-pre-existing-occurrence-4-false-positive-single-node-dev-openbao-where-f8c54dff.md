---
type: Incident
title: 'OpenBaoRaftQuorumAtRisk (pre-existing, occurrence #4) — false positive: single-node dev OpenBao where…'
description: The alert fires on vault_autopilot_failure_tolerance < 1, but this OpenBao deployment is a single-node dev instance where failure_tolerance=0 is the mathematically correct, inherent value — not a quorum degradation. The alert threshold is inappropriate for this topology.
resource: openbao/openbao-dev-cr28
tags:
    - runlore
    - incident
    - instance
    - openbao
timestamp: "2026-09-12T21:44:00Z"
fingerprint: f8c54dff1538e9c5fd89466cf382ad91af2d9d617098452f37b5e31530314bd4
confidence: 0.75
---

## Decision

- **why keep:** The alert fires on vault_autopilot_failure_tolerance < 1, but this OpenBao deployment is a single-node dev instance where failure_tolerance=0 is the mathematically correct, inherent value — not a quorum degradation. The alert threshold is inappropriate for this topology.
- **confidence:** 75%

## Symptom

OpenBaoRaftQuorumAtRisk (pre-existing, occurrence #4) — false positive: single-node dev OpenBao where failure_tolerance=0 is expected

Affected resource: Instance openbao/openbao-dev-cr28

## Investigate

- alert_rule: OpenBaoRaftQuorumAtRisk expr is `vault_autopilot_failure_tolerance{job="openbao"} < 1` for=15m — fires when failure_tolerance is below 1
- query_metrics: `vault_autopilot_failure_tolerance{job="openbao"} = 0` — the metric the rule thresholds is 0
- query_metrics: `count(vault_autopilot_node_healthy{job="openbao"}) = 1` — exactly ONE node in the raft cluster; for a single voter, floor((1-1)/2)=0 is the expected failure tolerance, not a sign of a lost peer
- query_metrics: `vault_autopilot_node_healthy{...node_id="openbao-dev-cr28.europe-west4-a.c.ogenki-435905.internal"} = 1` — the single node's ID literally contains 'openbao-dev' and it is healthy
- query_metrics: `vault_autopilot_healthy = 1` — the cluster reports healthy overall
- query_metrics_range over 31h: `vault_autopilot_node_healthy` has been flat at 1 and `vault_autopilot_failure_tolerance` has been flat at 0 for the entire window — there was never a second voter; this is steady-state, not a degradation from a previously larger cluster
- cloud_resource_health: GCE instance `openbao-dev-cr28` is status=RUNNING — the node is up, not dead
- query_metrics_range: `process_start_time_seconds{job="openbao"}` is flat at 1789138703 (2026-09-11T15:30:03Z) across the full 31h window — the OpenBao process has been running continuously, no crash or restart during the incident
- pod_status / list_resources StatefulSet: no pods and no StatefulSet exist in the `openbao` namespace — this is a standalone GCE instance (bao.priv.gcp.ogenki.io:8200), not a Kubernetes-managed HA workload

## Cause

1. **The alert fires on vault_autopilot_failure_tolerance < 1, but this OpenBao deployment is a single-node dev instance where failure_tolerance=0 is the mathematically correct, inherent value — not a quorum degradation. The alert threshold is inappropriate for this topology.** (75%)

## Resolution

- This is a known false-positive on a single-node dev topology. The durable fix is to either (a) add a `cluster`/`node_id` label override or a separate alert rule that suppresses OpenBaoRaftQuorumAtRisk and OpenBaoRaftNodeLost when `vault_autopilot_node_healthy` count is 1 (single-node), or (b) exclude this dev instance's scrape target from the openbao alert group. No operational action is needed on the instance itself — it is healthy. (reversible=true)

