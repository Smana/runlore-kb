---
type: Incident
title: 'Kustomization/agent-factory HealthCheckFailed (occurrence #3, pre-existing) — HelmRelease agent-factory still…'
description: 'The HelmRelease agent-system/agent-factory remains in a terminal Stalled=True (MissingRollbackTarget) state from the same failed upgrade first identified in occurrence #2. The upgrade to chart agent-factory@0.53.2-pr618.g9d0bdb48+312dd4d13e15 failed with ''context canceled'' at 15:00:01Z (version 2, status: failed), and the remediation rollback failed because there is no prior successful release to roll back to (''missing target release for rollback: cannot remediate failed release''). The Stalled condition (lastTransitionTime 2026-11-17T15:00:12Z) blocks the helm-controller from attempting any new reconciliation — it has not processed agent-factory in 180+ minutes while all other HelmReleases reconcile normally. The Kustomization health check fails because it sees the HelmRelease status as ''Failed''. A `flux reconcile helmrelease agent-factory -n agent-system --reset` is required to clear the Stalled/failure state and force a fresh install/upgrade attempt.'
resource: agent-system/agent-factory
alert_resource: flux-system/agent-factory
tags:
    - runlore
    - incident
    - helmrelease
    - agent-system
timestamp: "2026-11-17T16:11:19Z"
fingerprint: 0e9430b8bb5bbb3c6ae225f9b354d8326071c7d0a4a676ac401543d445e0e7d5
confidence: 0.8
provenance:
    - 'No new GitOps or cloud change — same pre-existing Stalled state from occurrence #2. what_changed(flux-system/agent-factory) shows revision b91b814e synced at 16:08:46Z but all SSA outputs are ''unchanged''.'
origin: runlore
---

## Decision

- **why keep:** The HelmRelease agent-system/agent-factory remains in a terminal Stalled=True (MissingRollbackTarget) state from the same failed upgrade first identified in occurrence #2. The upgrade to chart agent-factory@0.53.2-pr618.g9d0bdb48+312dd4d13e15 failed with 'context canceled' at 15:00:01Z (version 2, status: failed), and the remediation rollback failed because there is no prior successful release to roll back to ('missing target release for rollback: cannot remediate failed release'). The Stalled condition (lastTransitionTime 2026-11-17T15:00:12Z) blocks the helm-controller from attempting any new reconciliation — it has not processed agent-factory in 180+ minutes while all other HelmReleases reconcile normally. The Kustomization health check fails because it sees the HelmRelease status as 'Failed'. A `flux reconcile helmrelease agent-factory -n agent-system --reset` is required to clear the Stalled/failure state and force a fresh install/upgrade attempt.
- **confidence:** 80%
- **provenance:** No new GitOps or cloud change — same pre-existing Stalled state from occurrence #2. what_changed(flux-system/agent-factory) shows revision b91b814e synced at 16:08:46Z but all SSA outputs are 'unchanged'.

## Symptom

Kustomization/agent-factory HealthCheckFailed (occurrence #3, pre-existing) — HelmRelease agent-factory still Stalled (MissingRollbackTarget) after failed upgrade; pods healthy

Affected resource: HelmRelease agent-system/agent-factory

## Investigate

- resource_spec(HelmRelease/agent-system/agent-factory): status.conditions Stalled=True, reason=MissingRollbackTarget, message='Failed to perform remediation: missing target release for rollback: cannot remediate failed release', lastTransitionTime=2026-11-17T15:00:12Z
- resource_spec(HelmRelease/agent-system/agent-factory): status.history shows 2 failed upgrades (version 1 and 2) of chart agent-factory@0.53.2-pr618.g9d0bdb48+312dd4d13e15, both status=failed; version 2 lastDeployed=2026-11-17T15:00:01Z
- resource_spec(HelmRelease/agent-system/agent-factory): status.conditions Ready=False, reason=HealthCheckCanceled, message='New reconciliation triggered by ConfigMap/agent-system/agent-factory-helm-values-sponsored' — a new reconcile was attempted but blocked by Stalled
- resource_spec(HelmRelease/agent-system/agent-factory): status.conditions Released=False, reason=UpgradeFailed, message='Helm upgrade failed for release agent-system/agent-factory with chart agent-factory@0.53.2-pr618.g9d0bdb48+312dd4d13e15: context canceled'
- controller_logs(helm-controller, 180m): NO entries for agent-system/agent-factory — only agent-factory-crds appears as 'in-sync with desired state' at 15:57:07Z; the Stalled condition has paused reconciliation of the main HelmRelease
- controller_logs(kustomize-controller, agent-factory): SSA completes successfully with all resources 'unchanged' at 16:07-16:09, then immediately fails health check: 'health check failed after 33.363337ms: failed early due to stalled resources: [HelmRelease/agent-system/agent-factory status: Failed]'
- pod_status(agent-system): agent-factory-74f687df9-frhcn and -qlbxn both Running ready=1/1 age=1h — pods are healthy, no pod-level issue
- query_metrics(kube_deployment_status_condition): deployment=agent-factory Available=True, reason=MinimumReplicasAvailable — workload is serving traffic
- gitops_tree(Kustomization/agent-factory): all dependencies (agent-factory-crds, agent-secrets, room-broker, agent-policies) Ready=True — failure is isolated to the HelmRelease itself

## Cause

1. **The HelmRelease agent-system/agent-factory remains in a terminal Stalled=True (MissingRollbackTarget) state from the same failed upgrade first identified in occurrence #2. The upgrade to chart agent-factory@0.53.2-pr618.g9d0bdb48+312dd4d13e15 failed with 'context canceled' at 15:00:01Z (version 2, status: failed), and the remediation rollback failed because there is no prior successful release to roll back to ('missing target release for rollback: cannot remediate failed release'). The Stalled condition (lastTransitionTime 2026-11-17T15:00:12Z) blocks the helm-controller from attempting any new reconciliation — it has not processed agent-factory in 180+ minutes while all other HelmReleases reconcile normally. The Kustomization health check fails because it sees the HelmRelease status as 'Failed'. A `flux reconcile helmrelease agent-factory -n agent-system --reset` is required to clear the Stalled/failure state and force a fresh install/upgrade attempt.** (80%) — change: No new GitOps or cloud change — same pre-existing Stalled state from occurrence #2. what_changed(flux-system/agent-factory) shows revision b91b814e synced at 16:08:46Z but all SSA outputs are 'unchanged'.

## Resolution

- No action suggested by the investigation.

## Citations

[1] No new GitOps or cloud change — same pre-existing Stalled state from occurrence #2. what_changed(flux-system/agent-factory) shows revision b91b814e synced at 16:08:46Z but all SSA outputs are 'unchanged'.

