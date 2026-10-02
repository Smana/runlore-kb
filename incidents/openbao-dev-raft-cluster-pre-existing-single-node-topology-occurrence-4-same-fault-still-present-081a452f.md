---
type: Incident
title: 'OpenBao dev raft cluster — pre-existing single-node topology (occurrence #4, same fault still present)'
description: The OpenBao dev cluster is a single-node raft deployment — failure tolerance is structurally 0 (one node = zero redundancy), which is below the alert threshold of 1. This is a standing topology condition, NOT a node-loss event. The instance is healthy and has been continuously up for the entire 2-day alert window.
resource: security/openbao
tags:
    - runlore
    - incident
    - service
    - security
timestamp: "2026-10-02T19:16:16Z"
fingerprint: 081a452fe20e994635fe376498690a32672356c3d6fbba2f651922b0b4424e80
confidence: 0.9
---

## Decision

- **why keep:** The OpenBao dev cluster is a single-node raft deployment — failure tolerance is structurally 0 (one node = zero redundancy), which is below the alert threshold of 1. This is a standing topology condition, NOT a node-loss event. The instance is healthy and has been continuously up for the entire 2-day alert window.
- **confidence:** 90%

## Symptom

OpenBao dev raft cluster — pre-existing single-node topology (occurrence #4, same fault still present)

Affected resource: Service security/openbao

## Investigate

- alert_rule OpenBaoRaftQuorumAtRisk: expr is vault_autopilot_failure_tolerance{job="openbao"} < 1; the metric's current value is 0, so the alert fires — but 0 is the mathematically correct failure tolerance for any 1-node raft cluster.
- query_metrics vault_autopilot_failure_tolerance = 0; vault_autopilot_healthy = 1 (cluster healthy); vault_autopilot_node_healthy = 1 (the single node healthy); only ONE node_id series exists: openbao-dev-t73s.europe-west4-a.c.ogenki-435905.internal.
- query_metrics count(up{job="openbao"}) = 1 — only one scrape target exists. discover_metrics for job="openbao" confirms a single instance: bao.priv.gcp.ogenki.io:8200.
- query_metrics_range over 2880 minutes: vault_autopilot_failure_tolerance is flat at 0 (first=0, last=0, min=0, max=0); vault_autopilot_healthy flat at 1; up flat at 1; process_start_time_seconds flat (no restart). The alert ALERTS{OpenBaoRaftQuorumAtRisk,alertstate=firing} has been continuously firing for the full 2-day window — this is a standing condition, not an acute event.
- resource_spec Service security/openbao: type=ExternalName, externalName=bao.priv.gcp.ogenki.io — OpenBao runs on an external Compute Engine VM, not as a Kubernetes pod. cloud_resource_health: compute instance openbao-dev-t73s status=RUNNING, last started 2026-09-30T11:07:37 (before the incident window).
- The companion alert OpenBaoRaftNodeLost (expr: vault_autopilot_failure_tolerance < 2) is also firing at value 0 — consistent with single-node topology, not a dead peer. The alert annotation mentions 'instances replaced without bao operator raft remove-peer stay in the voter set' and 'spot reclamation,' but there is no evidence of a dead/stale peer: only one node_healthy series exists with value 1, and no dead-voter metrics are present.

## Cause

1. **The OpenBao dev cluster is a single-node raft deployment — failure tolerance is structurally 0 (one node = zero redundancy), which is below the alert threshold of 1. This is a standing topology condition, NOT a node-loss event. The instance is healthy and has been continuously up for the entire 2-day alert window.** (90%)

## Resolution

- This is a known dev-environment topology choice, not an acute failure. Two options: (1) If single-node is acceptable for dev, silence or re-tune these alerts for the dev environment so they stop paging — the cluster is healthy. (2) If HA is required even in dev, scale OpenBao to 3 raft peers (the external VM deployment, not Kubernetes-side), which would raise failure tolerance to 1 and clear both OpenBaoRaftQuorumAtRisk and OpenBaoRaftNodeLost. The Kubernetes-side GitOps (security-openbao Kustomization managing Service/ClusterIssuer/ClusterSecretStore) is healthy and needs no action. (reversible=true)

## Unresolved

- Whether single-node OpenBao is an intentional/accepted dev topology or an oversight that should be remediated to 3 nodes — this is a decision for the platform team. The alert is technically correct (a 1-node cluster cannot tolerate a failure) but may be noise for a dev environment.

