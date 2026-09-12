---
type: Incident
title: 'OpenBaoRaftQuorumAtRisk (pre-existing, occurrence #5) — false positive: single-node dev OpenBao where…'
description: Single-node dev OpenBao cluster reports failure_tolerance=0, which is the mathematically correct and expected value for N=1 (floor((1-1)/2)=0), not a lost-quorum condition. The alert rule `vault_autopilot_failure_tolerance < 1` does not account for single-node dev deployments where 0 is the normal steady state.
resource: external-vm/openbao-dev-cr28
tags:
    - runlore
    - incident
    - computeinstance
    - external-vm
timestamp: "2026-09-12T22:34:01Z"
fingerprint: 96dae7456a4926216419411a85df78ed190ad6aac9a3cce6642b55f84a245328
confidence: 0.9
---

## Decision

- **why keep:** Single-node dev OpenBao cluster reports failure_tolerance=0, which is the mathematically correct and expected value for N=1 (floor((1-1)/2)=0), not a lost-quorum condition. The alert rule `vault_autopilot_failure_tolerance < 1` does not account for single-node dev deployments where 0 is the normal steady state.
- **confidence:** 90%

## Symptom

OpenBaoRaftQuorumAtRisk (pre-existing, occurrence #5) — false positive: single-node dev OpenBao where failure_tolerance=0 is expected

Affected resource: ComputeInstance external-vm/openbao-dev-cr28

## Investigate

- alert_rule: expr=`vault_autopilot_failure_tolerance{job="openbao"} < 1` for=15m — the rule fires whenever failure_tolerance is 0, with no condition distinguishing a real quorum loss from a single-node dev cluster.
- query_metrics: `vault_autopilot_failure_tolerance{job="openbao"} = 0` — exactly one series, value 0.
- query_metrics: `vault_autopilot_node_healthy{job="openbao"}` returns exactly ONE node: `openbao-dev-cr28.europe-west4-a.c.ogenki-435905.internal` = 1 (healthy). There are no other nodes, healthy or unhealthy.
- query_metrics_range: `vault_autopilot_node_healthy` trend is flat at 1 for the entire 30m window (1>1>1>...>1), and `vault_autopilot_failure_tolerance` is flat at 0 (0>0>0>...>0) — no step-change, no degradation, no node loss event.
- query_metrics: the `vault_autopilot_node_healthy == 0` subquery (unhealthy/lost nodes) returned NO series — no peer was lost, so OpenBaoRaftNodeLost is not firing and the alert's own annotation about 'that same dead peer plus a real one' does not apply.
- cloud_resource_health: compute instance `openbao-dev-cr28` status=RUNNING, last started 2026-09-11 (yesterday) — no instance churn.
- query_metrics_range: `up{job="openbao"}` is flat at 1 for the entire window — scraping is uninterrupted.
- pod_status namespace=openbao: no pods — the OpenBao instance `bao.priv.gcp.ogenki.io:8200` is an external VM, not a Kubernetes workload, so there is no GitOps change or pod-level fault to trace.
- Alert labels: env="dev", node_id contains "openbao-dev-" — this is a development environment.

## Cause

1. **Single-node dev OpenBao cluster reports failure_tolerance=0, which is the mathematically correct and expected value for N=1 (floor((1-1)/2)=0), not a lost-quorum condition. The alert rule `vault_autopilot_failure_tolerance < 1` does not account for single-node dev deployments where 0 is the normal steady state.** (90%)

## Resolution

- Adjust the alert rule so it does not fire on expected single-node states. Either (a) scope `OpenBaoRaftQuorumAtRisk` to non-dev environments (e.g. add `env!="dev"`), or (b) make the expression require an actual unhealthy peer: `vault_autopilot_failure_tolerance{job="openbao"} < 1 and on(instance) (vault_autopilot_node_healthy{job="openbao"} == 0)`. For a single-node dev cluster, failure_tolerance=0 is by design and losing that one node is a VM-down problem, not a raft quorum problem. (reversible=true)

## Unresolved

- Whether the alert rule owners intend OpenBao dev to run as a single-node cluster permanently or plan to scale it to 3 nodes (which would make failure_tolerance=1 and silence the alert organically). This is a design intent question only a human can answer.

