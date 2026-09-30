---
type: Incident
title: OpenBao dev raft cluster at 1 peer / 0 failure tolerance — freshly provisioned single-node cluster
description: OpenBao dev cluster was freshly provisioned at ~18:07Z with only a single node, resulting in a 1-peer raft cluster with 0 failure tolerance. The entire GCP infrastructure stack (instance template, instance group manager, compute instance openbao-dev-t73s, backend service, forwarding rule, health check, firewall rules, KMS unseal key, reserved IP) was created from scratch by user smaine.kahlouch@ogenki.io between 18:07:09Z and 18:08:00Z. Only one compute instance exists. The raft cluster reports vault_raft_peers=1, vault_autopilot_failure_tolerance=0, and only one healthy node (openbao-dev-t73s.europe-west4-a.c.ogenki-435905.internal). A 1-node cluster mathematically cannot tolerate any failure, so both OpenBaoRaftQuorumAtRisk (FT<1) and OpenBaoRaftNodeLost (FT<2) fire correctly. The process started at ~18:08:50Z; metrics first appeared at ~18:26Z when the scrape config picked up the new endpoint; the 15m hold expired at ~18:41Z matching the incident start time.
resource: security/openbao-dev-t73s
tags:
    - runlore
    - incident
    - compute instance
    - security
timestamp: "2026-09-30T19:16:07Z"
fingerprint: 074f39c623f4740f356d6990858fa577e846f6ab8a1ad6ae9a49ad9263389abb
confidence: 0.8
provenance:
    - 'GCP Cloud Audit Log: compute.instances.insert for openbao-dev-t73s at 2026-09-30T18:07:28Z by 323586397743@cloudservices.gserviceaccount.com (triggered by smaine.kahlouch@ogenki.io creating the IGM at 18:07:12Z)'
---

## Decision

- **why keep:** OpenBao dev cluster was freshly provisioned at ~18:07Z with only a single node, resulting in a 1-peer raft cluster with 0 failure tolerance. The entire GCP infrastructure stack (instance template, instance group manager, compute instance openbao-dev-t73s, backend service, forwarding rule, health check, firewall rules, KMS unseal key, reserved IP) was created from scratch by user smaine.kahlouch@ogenki.io between 18:07:09Z and 18:08:00Z. Only one compute instance exists. The raft cluster reports vault_raft_peers=1, vault_autopilot_failure_tolerance=0, and only one healthy node (openbao-dev-t73s.europe-west4-a.c.ogenki-435905.internal). A 1-node cluster mathematically cannot tolerate any failure, so both OpenBaoRaftQuorumAtRisk (FT<1) and OpenBaoRaftNodeLost (FT<2) fire correctly. The process started at ~18:08:50Z; metrics first appeared at ~18:26Z when the scrape config picked up the new endpoint; the 15m hold expired at ~18:41Z matching the incident start time.
- **confidence:** 80%
- **provenance:** GCP Cloud Audit Log: compute.instances.insert for openbao-dev-t73s at 2026-09-30T18:07:28Z by 323586397743@cloudservices.gserviceaccount.com (triggered by smaine.kahlouch@ogenki.io creating the IGM at 18:07:12Z)

## Symptom

OpenBao dev raft cluster at 1 peer / 0 failure tolerance — freshly provisioned single-node cluster

Affected resource: Compute Instance security/openbao-dev-t73s

## Investigate

- cloud_what_changed(resource=openbao-dev, since_minutes=180): CREATE operations by smaine.kahlouch@ogenki.io at 18:07:09–18:08:00Z for instance template openbao-dev-20260930180709511800000001, instance group manager openbao-dev, instance group openbao-dev, compute instance openbao-dev-t73s, backend service openbao-dev, forwarding rule openbao-dev, health check openbao-dev, firewall rules openbao-dev-api/openbao-dev-health-check, KMS crypto key openbao-unseal, reserved IP openbao-dev — all insert operations, a full fresh provisioning
- query_metrics: vault_autopilot_failure_tolerance{job="openbao"} = 0 (the exact metric the alert thresholds)
- query_metrics: vault_raft_peers{job="openbao"} = 1 — only one raft peer exists, not the 5 the NodeLost alert description references
- query_metrics: vault_autopilot_node_healthy shows only one node_id: openbao-dev-t73s.europe-west4-a.c.ogenki-435905.internal
- query_metrics: process_start_time_seconds{job="openbao"} = 1790791730.5 → 2026-09-30T18:08:50.5Z, consistent with the instance creation at 18:07:37Z
- query_metrics_range: vault_autopilot_failure_tolerance has been 0 for the entire observed window (first data point 18:26Z), and up=1 throughout — the target is alive and healthy, just alone
- cloud_resource_health(instance_id=openbao-dev-t73s): status=RUNNING, last started 2026-09-30T11:07:37 PDT (=18:07:37 UTC)
- resource_spec(Service/security/openbao): ExternalName service pointing to bao.priv.gcp.ogenki.io:8200 — OpenBao runs on GCP VMs, not in Kubernetes

## Cause

1. **OpenBao dev cluster was freshly provisioned at ~18:07Z with only a single node, resulting in a 1-peer raft cluster with 0 failure tolerance. The entire GCP infrastructure stack (instance template, instance group manager, compute instance openbao-dev-t73s, backend service, forwarding rule, health check, firewall rules, KMS unseal key, reserved IP) was created from scratch by user smaine.kahlouch@ogenki.io between 18:07:09Z and 18:08:00Z. Only one compute instance exists. The raft cluster reports vault_raft_peers=1, vault_autopilot_failure_tolerance=0, and only one healthy node (openbao-dev-t73s.europe-west4-a.c.ogenki-435905.internal). A 1-node cluster mathematically cannot tolerate any failure, so both OpenBaoRaftQuorumAtRisk (FT<1) and OpenBaoRaftNodeLost (FT<2) fire correctly. The process started at ~18:08:50Z; metrics first appeared at ~18:26Z when the scrape config picked up the new endpoint; the 15m hold expired at ~18:41Z matching the incident start time.** (80%) — change: GCP Cloud Audit Log: compute.instances.insert for openbao-dev-t73s at 2026-09-30T18:07:28Z by 323586397743@cloudservices.gserviceaccount.com (triggered by smaine.kahlouch@ogenki.io creating the IGM at 18:07:12Z)

## Resolution

- Determine whether the dev OpenBao cluster is intended to be single-node or multi-node. If multi-node (the alerts assume 5): scale the instance group manager openbao-dev to the target size and join new nodes via `bao operator raft add-peer`. If single-node is intentional for dev: silence/adjust the OpenBaoRaftQuorumAtRisk and OpenBaoRaftNodeLost alerts for env=dev, since a 1-node cluster will always have FT=0. In either case, this is not a spot-reclamation/stale-voter scenario (vault_raft_peers=1 rules that out) — the runbook's `bao operator raft remove-peer` remedy does not apply here. (reversible=true)

## Unresolved

- Whether the single-node configuration is intentional for the dev environment (in which case the alerts should be silenced/adjusted for env=dev) or an incomplete provisioning of what should be a multi-node cluster. This requires checking the IGM target size or consulting the provisioning intent with smaine.kahlouch@ogenki.io who created the infrastructure.

## Citations

[1] GCP Cloud Audit Log: compute.instances.insert for openbao-dev-t73s at 2026-09-30T18:07:28Z by 323586397743@cloudservices.gserviceaccount.com (triggered by smaine.kahlouch@ogenki.io creating the IGM at 18:07:12Z)

