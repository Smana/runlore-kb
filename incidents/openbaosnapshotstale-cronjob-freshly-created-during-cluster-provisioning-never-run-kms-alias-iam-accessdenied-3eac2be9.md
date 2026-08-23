---
type: Incident
title: OpenBaoSnapshotStale — CronJob freshly created during cluster provisioning, never run; KMS Alias IAM AccessDenied…
description: The openbao-snapshot CronJob was freshly created at ~2026-08-23T18:02Z as part of a cluster provisioning event and has never reached its first scheduled execution (~04:00Z tomorrow, a daily schedule). The alert condition 'CronJob has never succeeded once' is met immediately upon creation, triggering the critical alert ~4 minutes later. The CronJob is not suspended (spec_suspend=0), has status_active=0, no job events, no pods, no logs, and no kube_job_status_successful/failed metrics exist for it at all — confirming it has simply never run yet, not that it ran and failed.
resource: security/openbao-snapshot
tags:
    - runlore
    - incident
    - cronjob
    - security
timestamp: "2026-08-23T19:19:31Z"
fingerprint: 3eac2be9b3075e179e57042f420087a91e565a479e974beadb79c75482f8bdef
confidence: 0.8
provenance:
    - Cluster provisioning / Flux Kustomization security applied CronJob/security/openbao-snapshot at ~2026-08-23T18:02Z (revision latest@sha256:0f48a6f47bf34a85843cb38397991335b5c9554fe05c49cffd538c18f7cc4feb)
---

## Decision

- **why keep:** The openbao-snapshot CronJob was freshly created at ~2026-08-23T18:02Z as part of a cluster provisioning event and has never reached its first scheduled execution (~04:00Z tomorrow, a daily schedule). The alert condition 'CronJob has never succeeded once' is met immediately upon creation, triggering the critical alert ~4 minutes later. The CronJob is not suspended (spec_suspend=0), has status_active=0, no job events, no pods, no logs, and no kube_job_status_successful/failed metrics exist for it at all — confirming it has simply never run yet, not that it ran and failed.
- **confidence:** 80%
- **provenance:** Cluster provisioning / Flux Kustomization security applied CronJob/security/openbao-snapshot at ~2026-08-23T18:02Z (revision latest@sha256:0f48a6f47bf34a85843cb38397991335b5c9554fe05c49cffd538c18f7cc4feb)

## Symptom

OpenBaoSnapshotStale — CronJob freshly created during cluster provisioning, never run; KMS Alias IAM AccessDenied blocks snapshot encryption infra

Affected resource: CronJob security/openbao-snapshot

## Investigate

- kube_cronjob_created timestamp = 1787508148 ≈ 2026-08-23T18:02:28Z — just 4 min before alert pending at 18:07Z
- kube_cronjob_spec_suspend = 0 (not suspended)
- kube_cronjob_status_active = 0 throughout entire observed window (18:07-19:12Z)
- kube_cronjob_next_schedule_time = 1787544000 ≈ 2026-08-24T04:00Z — next run is ~10 hours away
- kube_events for object 'openbao-snapshot' returned 'no events in namespace security' — no Job creation, no pod scheduling, no failures
- kube_job_status_succeeded and kube_job_status_failed metrics show only zitadel-init, zitadel-cleanup, and xplane-zitadel-cnpg-cluster-1-full-recovery jobs — none for openbao-snapshot
- kustomize-controller logs show CronJob/security/openbao-snapshot as 'unchanged' in the security Kustomization — the object exists and is applied, it just hasn't triggered
- pod_status with selector app=openbao-snapshot returned 'no pods in namespace security matching selector'

## Cause

1. **The openbao-snapshot CronJob was freshly created at ~2026-08-23T18:02Z as part of a cluster provisioning event and has never reached its first scheduled execution (~04:00Z tomorrow, a daily schedule). The alert condition 'CronJob has never succeeded once' is met immediately upon creation, triggering the critical alert ~4 minutes later. The CronJob is not suspended (spec_suspend=0), has status_active=0, no job events, no pods, no logs, and no kube_job_status_successful/failed metrics exist for it at all — confirming it has simply never run yet, not that it ran and failed.** (85%) — change: Cluster provisioning / Flux Kustomization security applied CronJob/security/openbao-snapshot at ~2026-08-23T18:02Z (revision latest@sha256:0f48a6f47bf34a85843cb38397991335b5c9554fe05c49cffd538c18f7cc4feb)

## Resolution

- Wait for the next scheduled CronJob execution (~04:00Z). If the alert should not fire during initial provisioning, consider adding a grace period to the alert rule (e.g., suppress for the first 24h after CronJob creation). (reversible=true)

## Unresolved

- Whether the KMS Alias AccessDenied will actually prevent the snapshot from succeeding when the CronJob runs at ~04:00Z — the Key resource exists ('unchanged') and the bucket SSE config succeeded, so the snapshot may use the key directly by ARN rather than through the alias. This can only be confirmed when the CronJob actually executes.
- Whether the Cilium IPAM exhaustion ('all CIDR ranges are exhausted') has fully resolved or will recur — it was observed at 18:23-18:27Z for zitadel-init pods, and later pods scheduled successfully, but the root cause of IPAM pool exhaustion on a 5-node cluster is unclear

## Citations

[1] Cluster provisioning / Flux Kustomization security applied CronJob/security/openbao-snapshot at ~2026-08-23T18:02Z (revision latest@sha256:0f48a6f47bf34a85843cb38397991335b5c9554fe05c49cffd538c18f7cc4feb)

