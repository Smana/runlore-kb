---
type: Playbook
title: 'OpenBaoRaftQuorumAtRisk (pre-existing) — false positive: 5-node alert thresholds still applied to single-node dev…'
description: Alert rules with 5-node-cluster thresholds (failure_tolerance < 1 for QuorumAtRisk, < 2 for NodeLost) are applied to a freshly-provisioned single-node dev OpenBao cluster, where failure_tolerance is permanently 0 by design — not a real quorum loss. This is the same pre-existing fault as the previous occurrence (8h54m ago); it was not fixed.
tags:
    - runlore
    - playbook
timestamp: "2026-09-12T15:48:45Z"
fingerprint: 3d8abc14e75e4664bea2fe8645255f5b68eda97851584729b332c264bb3f4572
confidence: 0.82
---

## Decision

- **why keep:** Alert rules with 5-node-cluster thresholds (failure_tolerance < 1 for QuorumAtRisk, < 2 for NodeLost) are applied to a freshly-provisioned single-node dev OpenBao cluster, where failure_tolerance is permanently 0 by design — not a real quorum loss. This is the same pre-existing fault as the previous occurrence (8h54m ago); it was not fixed.
- **confidence:** 82%

## Symptom

OpenBaoRaftQuorumAtRisk (pre-existing) — false positive: 5-node alert thresholds still applied to single-node dev OpenBao cluster

## Investigate

- alert_rule for OpenBaoRaftQuorumAtRisk: expr='vault_autopilot_failure_tolerance{job="openbao"} < 1' for=15m, state=firing. The companion OpenBaoRaftNodeLost fires at '< 2' and its annotation explicitly states 'a five-node cluster is no longer tolerating the two failures it is sized for' — thresholds are sized for a 5-node cluster.
- query_metrics: vault_autopilot_failure_tolerance{job="openbao"} = 0 (the single series for instance bao.priv.gcp.ogenki.io:8200). A 1-node raft cluster has FT = N - quorum = 1 - 1 = 0 by definition.
- query_metrics_range over 1500m: failure_tolerance is flat at 0 for the entire window (trend=0>0>0>...>0, min=0, max=0, biggest jump +0). This is not a degradation from a higher value — it has always been 0.
- query_metrics: count(count by (peer_id)(vault_raft_storage_stats_applied_index{job="openbao"})) = 1 — only one raft peer exists: peer_id='openbao-dev-cr28.europe-west4-a.c.ogenki-435905.internal' (a single GCE VM, name contains 'dev').
- query_metrics: vault_autopilot_healthy{job="openbao"} = 1 and up{job="openbao"} = 1 — the cluster reports itself as healthy and is reachable. No dead voters or degraded nodes.
- cloud_what_changed: the entire openbao-dev infrastructure (instance template, IGM, instance openbao-dev-cr28, firewall rules, backend service, forwarding rule, KMS unseal key) was freshly provisioned at 2026-09-11T14:56-14:57Z by smaine.kahlouch@ogenki.io — 44 minutes before the alert fired at 15:40:40Z. The alert fired after the 15m hold-down once metrics scraping began.
- cloud_resource_health: compute instance openbao-dev-cr28 status=RUNNING. No failed GCE provisioning events for openbao-dev (cloud_what_changed failed_only=true returned no failures for this resource).

## Cause

1. **Alert rules with 5-node-cluster thresholds (failure_tolerance < 1 for QuorumAtRisk, < 2 for NodeLost) are applied to a freshly-provisioned single-node dev OpenBao cluster, where failure_tolerance is permanently 0 by design — not a real quorum loss. This is the same pre-existing fault as the previous occurrence (8h54m ago); it was not fixed.** (82%)

## Resolution

- Fix the alerting rules so 5-node thresholds do not fire on single-node dev clusters. Options: (1) Add a label/relabeled env dimension (e.g., env=dev) to the openbao job and gate the alert with {env!="dev"} or use a separate lower-threshold rule for dev; (2) make the threshold relative to cluster size (e.g., failure_tolerance < floor((num_servers - 1) / 2)); (3) if this dev instance is intentionally single-node, suppress OpenBaoRaftQuorumAtRisk and OpenBaoRaftNodeLost for it. No action is needed on the OpenBao instance itself — it is healthy. (reversible=true)

## Unresolved

- Whether the openbao-dev single-node cluster is intended to remain single-node or should eventually be scaled to a multi-node raft cluster (at which point the existing thresholds would become correct). This is an architecture/owner decision.

