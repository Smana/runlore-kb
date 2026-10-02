---
type: Incident
title: 'OpenBao dev raft cluster — pre-existing single-node topology (occurrence #3, same fault still present)'
description: 'OpenBao dev raft cluster runs as a single external node, giving 0 failure tolerance by definition — the same pre-existing fault as occurrences #1 and #2, unchanged. The alert fires because the rule (vault_autopilot_failure_tolerance < 1) is designed for multi-node HA clusters, but this dev cluster has only one peer.'
resource: openbao/openbao-dev-t73s
tags:
    - runlore
    - incident
    - computeinstance
    - openbao
timestamp: "2026-10-02T07:18:05Z"
fingerprint: 230e714a9c3c0a3d9e4bd5e0ed1fd28014b6f3c35a363dbd411285bc78f53871
confidence: 0.88
---

## Decision

- **why keep:** OpenBao dev raft cluster runs as a single external node, giving 0 failure tolerance by definition — the same pre-existing fault as occurrences #1 and #2, unchanged. The alert fires because the rule (vault_autopilot_failure_tolerance < 1) is designed for multi-node HA clusters, but this dev cluster has only one peer.
- **confidence:** 88%

## Symptom

OpenBao dev raft cluster — pre-existing single-node topology (occurrence #3, same fault still present)

Affected resource: ComputeInstance openbao/openbao-dev-t73s

## Investigate

- alert_rule: OpenBaoRaftQuorumAtRisk expr is `vault_autopilot_failure_tolerance{job="openbao"} < 1` for=15m, state=firing — the metric value IS 0, so the alert correctly fires.
- query_metrics: `vault_autopilot_failure_tolerance{job="openbao"} = 0` — confirmed right now.
- query_metrics_range (2880m/48h lookback): `vault_autopilot_failure_tolerance` first=0 last=0 min=0 max=0 — flat at 0 for the ENTIRE 48-hour window. This is structural, not a recent degradation. No node was lost.
- query_metrics: `count by (peer_id) (vault_raft_storage_stats_applied_index{job="openbao"})` = 1, with peer_id=`openbao-dev-t73s.europe-west4-a.c.ogenki-435905.internal` — only ONE raft peer exists.
- query_metrics: `count by (instance) (up{job="openbao"})` = 1 for instance=`bao.priv.gcp.ogenki.io:8200` — only one target is scraped, across the entire 48h lookback.
- query_metrics: `vault_autopilot_node_healthy{job="openbao"}` returns exactly one series (node_id=openbao-dev-t73s...) = 1 — one healthy node, zero others.
- resource_spec VMScrapeConfig observability/openbao: staticConfigs has exactly one target (`bao.priv.gcp.ogenki.io:8200`) — deliberately scraping a single external instance, not a Kubernetes workload.
- cloud_resource_health: `compute instance openbao-dev-t73s (europe-west4-a): status=RUNNING` — OpenBao runs as an external GCP compute VM, not a K8s pod. No pods exist in the `openbao` namespace (namespace itself is ABSENT).
- query_metrics: `vault_autopilot_healthy{job="openbao"} = 1` and `up{job="openbao"} = 1` — the cluster IS operational (has a leader, serving requests). It's not degraded, just single-node.
- Previous investigation (occurrence #2, ~12h ago): same conclusion — 'OpenBao dev raft cluster at 1 peer / 0 failure tolerance — pre-existing single-node cluster (same fault still present)'. Live state confirms nothing has changed.
- alert_rule: OpenBaoRaftNodeLost expr is `vault_autopilot_failure_tolerance{job="openbao"} < 2` for=15m, state=firing — fires alongside the critical alert since 0 < 2.
- alert_rule annotation for OpenBaoRaftNodeLost: 'Failure tolerance has been below 2 for 15 minutes, so a five-node cluster is no longer tolerating the two failures it is sized for.' — the rule explicitly assumes a 5-node cluster, which this dev deployment is not.
- query_metrics: Both ALERTS_FOR_STATE series are present and firing: OpenBaoRaftNodeLost (severity=warning) and OpenBaoRaftQuorumAtRisk (severity=critical), both with the same start timestamp.

## Cause

1. **OpenBao dev raft cluster runs as a single external node, giving 0 failure tolerance by definition — the same pre-existing fault as occurrences #1 and #2, unchanged. The alert fires because the rule (vault_autopilot_failure_tolerance < 1) is designed for multi-node HA clusters, but this dev cluster has only one peer.** (88%)
2. **Both OpenBaoRaftNodeLost (warning, threshold < 2) and OpenBaoRaftQuorumAtRisk (critical, threshold < 1) fire simultaneously on a single-node cluster — the NodeLost alert's own annotation says it expects a 5-node cluster ('no longer tolerating the two failures it is sized for'), confirming a rule/env mismatch.** (55%)

## Resolution

- This is a dev environment running a single-node OpenBao by design. Two paths: (1) If single-node is acceptable for dev, suppress/silence OpenBaoRaftQuorumAtRisk and OpenBaoRaftNodeLost alerts for env=dev, or add an `env!="dev"` filter to the alert rule — the rules are sized for a multi-node HA cluster and will always fire here. (2) If HA is actually required even in dev, join 1-2 additional OpenBao peers to the raft cluster (each as a separate compute instance) and verify failure_tolerance rises to >=1. Path (1) is the pragmatic fix; path (2) is the resilience fix. (reversible=true)
- The alert rules should carry an env filter (e.g. env!="dev") or the dev cluster should be exempted via alert inhibition/silence rules, since these rules are designed for production multi-node topologies. (reversible=false)

## Unresolved

- Whether single-node OpenBao is the intended topology for the dev environment, or whether additional peers were planned but never provisioned — this is a design/policy question for the OpenBao service owner.
- Whether the process restart at ~2026-09-30T19:08:50Z (1 min before incident start) was a routine VM reboot or an unexpected crash — the GCP compute instance shows last started 2026-09-30T11:07:37-07:00 (18:07:37Z UTC), about an hour before the process start time. This gap may indicate a delayed OpenBao service start after VM boot, but it doesn't affect the root cause (single-node topology).

