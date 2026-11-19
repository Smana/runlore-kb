---
type: Incident
title: 'HelmRelease/agent-factory HealthCheckCanceled: pods couldn''t find servable config → install timeouts → ConfigMap…'
description: 'The agent-factory pods (ReplicaSet 6f98c995cc, image ghcr.io/smana/maku-factory:v0.53.2-pr618.9d0bdb48@sha256:8cd74cd52b0b) could not start because they were ''waiting for a servable config: apply the factory''s Factory, Repository and MergePolicy resources'' (×67 log entries across 9 pods) and got ''generation refused'' (×45). The readiness probe returned HTTP 503 continuously (kube_events Unhealthy ×25 per pod), so the Deployment never reached Ready during any of the three 5-minute Helm install windows (14:02, 14:22, 14:42), causing all three installs to time out with ''timeout waiting for: [Deployment/agent-system/agent-factory status: InProgress]''. The CRD HelmRelease (agent-factory-crds) successfully installed the CRDs (factories.maku.ogenki.io, repositories.maku.ogenki.io, mergepolicies.maku.ogenki.io) at 13:26, but the required CR instances (Factory, Repository) were not present in the cluster for the new app version to consume — the new image expected them to be applied and refused generation without them.'
resource: agent-system/agent-factory
tags:
    - runlore
    - incident
    - helmrelease
    - agent-system
timestamp: "2026-11-17T15:08:43Z"
fingerprint: 4865b90ae3012b0ec9c36d8701cc4781b55b3a28a8d2c7009c57d4d4a73bd91c
confidence: 0.82
provenance:
    - Image changed from sha256:ced3493d (v0.53.1) to sha256:8cd74cd5 (v0.53.2-pr618.9d0bdb48); chart agent-factory@0.53.2-pr618.g9d0bdb48+312dd4d13e15
origin: runlore
---

## Decision

- **why keep:** The agent-factory pods (ReplicaSet 6f98c995cc, image ghcr.io/smana/maku-factory:v0.53.2-pr618.9d0bdb48@sha256:8cd74cd52b0b) could not start because they were 'waiting for a servable config: apply the factory's Factory, Repository and MergePolicy resources' (×67 log entries across 9 pods) and got 'generation refused' (×45). The readiness probe returned HTTP 503 continuously (kube_events Unhealthy ×25 per pod), so the Deployment never reached Ready during any of the three 5-minute Helm install windows (14:02, 14:22, 14:42), causing all three installs to time out with 'timeout waiting for: [Deployment/agent-system/agent-factory status: InProgress]'. The CRD HelmRelease (agent-factory-crds) successfully installed the CRDs (factories.maku.ogenki.io, repositories.maku.ogenki.io, mergepolicies.maku.ogenki.io) at 13:26, but the required CR instances (Factory, Repository) were not present in the cluster for the new app version to consume — the new image expected them to be applied and refused generation without them.
- **confidence:** 82%
- **provenance:** Image changed from sha256:ced3493d (v0.53.1) to sha256:8cd74cd5 (v0.53.2-pr618.9d0bdb48); chart agent-factory@0.53.2-pr618.g9d0bdb48+312dd4d13e15

## Symptom

HelmRelease/agent-factory HealthCheckCanceled: pods couldn't find servable config → install timeouts → ConfigMap churn canceled upgrade → Stalled

Affected resource: HelmRelease agent-system/agent-factory

## Investigate

- query_logs(agent-factory, 90m): 'waiting for a servable config: apply the factory's Factory, Repository and MergePolicy resources' (×67 across 9 pods, first 13:56:20Z → last 15:00:11Z); 'generation refused' (×45, first 13:56:20Z → last 14:42:56Z); 'agent-factory stopped while waiting for a servable config' (×9)
- kube_events(agent-system, agent-factory, 120m): 'Unhealthy Pod/agent-factory-6f98c995cc-kkzfw (x25): Readiness probe failed: HTTP probe failed with statuscode: 503' — repeated for every pod in ReplicaSet 6f98c995cc
- gitops_resource_status(HelmRelease/agent-factory): three InstallFailed events at 14:07:28Z, 14:27:39Z, 14:47:51Z — each with 'timeout waiting for: [Deployment/agent-system/agent-factory status: InProgress]'
- query_metrics_range(kube_pod_container_status_ready{pod=~agent-factory-.*}): ReplicaSet 6f98c995cc pods (kkzfw, l4mr8, nb8b7, 67j7z, kftzx, sqxnd, etc.) all had ready=0 for their entire lifetime; ReplicaSet cfc8469b5 (old image sha256:ced3493d) had ready=1 throughout; ReplicaSet 74f687df9 (new) achieved ready=1 at 15:01:00Z
- resource_spec(HelmRelease/agent-factory-crds): CRDs installed successfully (UpgradeSucceeded at 13:26:52Z, v3, status=deployed) — CRDs are present but the required CR instances were not
- controller_logs(helm-controller, agent-factory, 90m): at 15:00:02.630Z 'New reconciliation triggered, canceling health checks' trigger='ConfigMap/agent-system/agent-factory-helm-values-sponsored'
- resource_spec(HelmRelease/agent-factory): status.conditions Ready=False (HealthCheckCanceled, 'New reconciliation triggered by ConfigMap/agent-system/agent-factory-helm-values-sponsored'); Released=False (UpgradeFailed, 'context canceled'); lastAttemptedReleaseActionDuration=928.273ms — the upgrade was canceled within a second
- resource_spec(HelmRelease/agent-factory): spec.valuesFrom includes both ConfigMap/agent-factory-helm-values and ConfigMap/agent-factory-helm-values-sponsored — the sponsored ConfigMap is a valuesFrom source that Flux watches
- resource_spec(HelmRelease/agent-factory): history v1 configDigest=sha256:2e5b5e2a31c53b69... vs v2 configDigest=sha256:a1ff5de1c6e7ddf3... — the values changed between the two attempts
- kube_events(agent-factory, 120m): at 15:00:02Z 'HealthCheckCanceled: Health checks canceled due to new reconciliation triggered by ConfigMap/agent-system/agent-factory-helm-values-sponsored'
- resource_spec(HelmRelease/agent-factory): status.conditions Stalled=True (MissingRollbackTarget, 'Failed to perform remediation: missing target release for rollback: cannot remediate failed release', lastTransitionTime=15:00:12Z)
- resource_spec(HelmRelease/agent-factory): history shows v1 status=failed (lastDeployed 14:42:50Z) and v2 status=failed (lastDeployed 15:00:01Z) — no successful release exists to roll back to
- controller_logs(helm-controller, agent-factory): at 15:00:12Z 'Reconciler Error ... terminal error: missing target release for rollback: cannot remediate failed release'
- resource_spec(Deployment/agent-factory): status.availableReplicas=2, readyReplicas=2, conditions Available=True (15:06:01Z), Progressing=True (NewReplicaSetAvailable 15:00:31Z) — pods recovered despite HelmRelease being Stalled
- kb_search runbook helmrelease-terminal-failed-exhausted-retries confirms: once retries/rollback are exhausted Flux stops retrying even after root cause is fixed; flux reconcile helmrelease --reset clears the counters

## Cause

1. **The agent-factory pods (ReplicaSet 6f98c995cc, image ghcr.io/smana/maku-factory:v0.53.2-pr618.9d0bdb48@sha256:8cd74cd52b0b) could not start because they were 'waiting for a servable config: apply the factory's Factory, Repository and MergePolicy resources' (×67 log entries across 9 pods) and got 'generation refused' (×45). The readiness probe returned HTTP 503 continuously (kube_events Unhealthy ×25 per pod), so the Deployment never reached Ready during any of the three 5-minute Helm install windows (14:02, 14:22, 14:42), causing all three installs to time out with 'timeout waiting for: [Deployment/agent-system/agent-factory status: InProgress]'. The CRD HelmRelease (agent-factory-crds) successfully installed the CRDs (factories.maku.ogenki.io, repositories.maku.ogenki.io, mergepolicies.maku.ogenki.io) at 13:26, but the required CR instances (Factory, Repository) were not present in the cluster for the new app version to consume — the new image expected them to be applied and refused generation without them.** (45%) — change: Image changed from sha256:ced3493d (v0.53.1) to sha256:8cd74cd5 (v0.53.2-pr618.9d0bdb48); chart agent-factory@0.53.2-pr618.g9d0bdb48+312dd4d13e15
2. **The ConfigMap agent-system/agent-factory-helm-values-sponsored (a valuesFrom source on the HelmRelease) was modified at ~15:00:02Z while an in-progress Helm upgrade was running (only 928ms into its execution). Flux detected the ConfigMap change and triggered a NEW reconciliation, canceling the in-progress upgrade's health checks and context. The upgrade failed with 'context canceled' before it could complete. This matches the known pattern documented in the KB runbook (incidents/helmrelease-envoy-ai-gateway-upgradefailed-externalsecret-password-generator-churn-on-valuesfrom-secret-cancels): a valuesFrom source that churns re-triggers the release it feeds, canceling in-progress operations. The configDigest changed between v1 (sha256:2e5b5e2a) and v2 (sha256:a1ff5de1), confirming the values changed.** (85%)
3. **The HelmRelease is now Stalled=True (MissingRollbackTarget, 'Failed to perform remediation: missing target release for rollback: cannot remediate failed release') because both Helm revisions (v1 and v2) are in status=failed — there is no prior successful release to roll back to. The upgrade.remediation.retries=3 with remediateLastFailure=true was attempted, but since no successful prior release exists, the rollback has no target. Flux will not retry on its own from this terminal state. The pods have actually recovered (ReplicaSet 74f687df9 is 2/2 ready since 15:06), but the HelmRelease remains Stalled and will not self-heal.** (82%)

## Resolution

- Clear the Stalled HelmRelease state: run `flux -n agent-system reconcile helmrelease agent-factory --reset` to reset the failure counters and trigger a fresh reconciliation. Since the pods are already running (ReplicaSet 74f687df9 is 2/2 ready), the reset should result in a successful release that marks the HelmRelease Ready. (reversible=true)

## Unresolved

- What process is writing to the ConfigMap/agent-system/agent-factory-helm-values-sponsored (causing it to churn and trigger new reconciliations)? RBAC denied reading the ConfigMap and listing MergePolicy resources, so the source of the ConfigMap mutation could not be identified. It may be a MergePolicy controller, an external automation, or a manual kubectl apply.
- Whether the required Factory/Repository CR instances were supposed to be created by the Helm chart itself (and the chart has a bug), or whether they are expected to be applied out-of-band (and someone forgot to apply them for the new app version). The pods' log says 'apply the factory's Factory, Repository and MergePolicy resources' but we could not read the chart templates to verify.
- Whether the pods recovered on their own (ReplicaSet 74f687df9 achieved 2/2 ready at 15:06) because the missing CR instances were eventually applied, or because the new ReplicaSet uses a different config that doesn't require them. The logs for the new ReplicaSet show 'agent-factory starting', 'Successfully acquired lease', 'Starting workers', 'merge decision' — indicating normal operation, but the exact trigger for recovery was not captured.

## Citations

[1] Image changed from sha256:ced3493d (v0.53.1) to sha256:8cd74cd5 (v0.53.2-pr618.9d0bdb48); chart agent-factory@0.53.2-pr618.g9d0bdb48+312dd4d13e15

