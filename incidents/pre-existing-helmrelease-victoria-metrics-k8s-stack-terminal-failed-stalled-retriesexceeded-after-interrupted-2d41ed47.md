---
type: Incident
title: '[PRE-EXISTING] HelmRelease victoria-metrics-k8s-stack terminal-failed (Stalled/RetriesExceeded) after interrupted…'
description: HelmRelease observability/victoria-metrics-k8s-stack is in a terminal Stalled state (RetriesExceeded) after an interrupted upgrade. At 15:14:17Z, a ConfigMap change (vm-common-helm-values) triggered a new reconciliation while a Helm upgrade to chart version 0.97.0 (release revision 5) was in progress, causing 'context canceled'. The interrupted upgrade was counted as 1 failure (status.failures=1). Because the HelmRelease spec has no upgrade.remediation.retries configured (Flux defaults to 0), a single upgrade failure immediately exhausted the retry budget — status condition Stalled=True with reason RetriesExceeded, message 'Failed to upgrade after 1 attempt(s)' at 15:14:28Z. Flux will not retry a Stalled HelmRelease on its own; the failure counters persist in status. The Kustomization flux-system/observability-victoria-metrics-k8s-stack health check is blocked because it depends on this HelmRelease being Ready, producing the recurring HealthCheckFailed alerts since 15:17:20Z. This is the same unfixed fault from the previous investigation 42 minutes ago — the HelmRelease has been Stalled since 15:14:28Z and no remediation has been applied.
resource: observability/victoria-metrics-k8s-stack
alert_resource: flux-system/observability-victoria-metrics-k8s-stack
tags:
    - runlore
    - incident
    - helmrelease
    - observability
timestamp: "2026-10-28T16:22:16Z"
fingerprint: 2d41ed47cbe06f81f93dba7b2e6cffb77f16fa2bbfddccfd933822d9d1d81691
confidence: 0.9
provenance:
    - ConfigMap/observability/vm-common-helm-values change at ~15:14:17Z interrupted in-flight Helm upgrade
---

## Decision

- **why keep:** HelmRelease observability/victoria-metrics-k8s-stack is in a terminal Stalled state (RetriesExceeded) after an interrupted upgrade. At 15:14:17Z, a ConfigMap change (vm-common-helm-values) triggered a new reconciliation while a Helm upgrade to chart version 0.97.0 (release revision 5) was in progress, causing 'context canceled'. The interrupted upgrade was counted as 1 failure (status.failures=1). Because the HelmRelease spec has no upgrade.remediation.retries configured (Flux defaults to 0), a single upgrade failure immediately exhausted the retry budget — status condition Stalled=True with reason RetriesExceeded, message 'Failed to upgrade after 1 attempt(s)' at 15:14:28Z. Flux will not retry a Stalled HelmRelease on its own; the failure counters persist in status. The Kustomization flux-system/observability-victoria-metrics-k8s-stack health check is blocked because it depends on this HelmRelease being Ready, producing the recurring HealthCheckFailed alerts since 15:17:20Z. This is the same unfixed fault from the previous investigation 42 minutes ago — the HelmRelease has been Stalled since 15:14:28Z and no remediation has been applied.
- **confidence:** 90%
- **provenance:** ConfigMap/observability/vm-common-helm-values change at ~15:14:17Z interrupted in-flight Helm upgrade

## Symptom

[PRE-EXISTING] HelmRelease victoria-metrics-k8s-stack terminal-failed (Stalled/RetriesExceeded) after interrupted upgrade — retries exhausted, Kustomization health check blocked (occurrence #2, unfixed)

Affected resource: HelmRelease observability/victoria-metrics-k8s-stack

## Investigate

- resource_spec HelmRelease observability/victoria-metrics-k8s-stack: Stalled=True (RetriesExceeded) 'Failed to upgrade after 1 attempt(s)' at 2026-10-28T15:14:28Z; Ready=False (HealthCheckCanceled) 'New reconciliation triggered by ConfigMap/observability/vm-common-helm-values' at 15:14:17Z; Released=False (UpgradeFailed) 'Helm upgrade failed for release observability/victoria-metrics-k8s-stack with chart victoria-metrics-k8s-stack@0.97.0: context canceled' at 15:14:17Z; status.failures=1
- resource_spec HelmRelease spec: install.strategy.name=RetryOnFailure with retryInterval=2m, but NO upgrade.remediation section configured — Flux defaults upgrade.remediation.retries to 0, so 1 failure = retries exhausted
- resource_spec HelmRelease history: revision 5 status=failed (lastDeployed 15:14:11Z), revision 4 status=deployed (lastDeployed 09:19:40Z) — the release storage is NOT wedged; a good deployed revision exists
- gitops_resource_status Kustomization flux-system/observability-victoria-metrics-k8s-stack: Ready=False (HealthCheckFailed) 'failed early due to stalled resources: [HelmRelease/observability/victoria-metrics-k8s-stack status: Failed]' — 13 HealthCheckFailed events from 15:17:20Z through 16:15:40Z
- controller_logs helm-controller (resource=observability, 180m): shows ALL other observability HelmReleases (victoria-logs, agent-traces-collector, grafana-operator, victoria-traces, crds-prometheus-operator) reconciling successfully every ~2-4m, but ZERO entries for victoria-metrics-k8s-stack — confirming it is Stalled and not being retried
- pod_status observability: ALL pods Running/Ready — vmsingle, vmagent, vmalert, vmalertmanager, grafana (3/3), kube-state-metrics, victoria-metrics-operator, node-exporter DaemonSet pods all healthy. The failed upgrade did not break running workloads; they continue on revision 4's config.
- kb_search returned runbook 'helmrelease-terminal-failed-exhausted-retries': 'Once retries are exhausted the HelmRelease enters a terminal state and Flux stops retrying, even after the root cause is fixed, because the failure counters persist in status. A plain flux reconcile and even flux suspend/resume do not clear it. Resolution: flux reconcile helmrelease --reset'

## Cause

1. **HelmRelease observability/victoria-metrics-k8s-stack is in a terminal Stalled state (RetriesExceeded) after an interrupted upgrade. At 15:14:17Z, a ConfigMap change (vm-common-helm-values) triggered a new reconciliation while a Helm upgrade to chart version 0.97.0 (release revision 5) was in progress, causing 'context canceled'. The interrupted upgrade was counted as 1 failure (status.failures=1). Because the HelmRelease spec has no upgrade.remediation.retries configured (Flux defaults to 0), a single upgrade failure immediately exhausted the retry budget — status condition Stalled=True with reason RetriesExceeded, message 'Failed to upgrade after 1 attempt(s)' at 15:14:28Z. Flux will not retry a Stalled HelmRelease on its own; the failure counters persist in status. The Kustomization flux-system/observability-victoria-metrics-k8s-stack health check is blocked because it depends on this HelmRelease being Ready, producing the recurring HealthCheckFailed alerts since 15:17:20Z. This is the same unfixed fault from the previous investigation 42 minutes ago — the HelmRelease has been Stalled since 15:14:28Z and no remediation has been applied.** (90%) — change: ConfigMap/observability/vm-common-helm-values change at ~15:14:17Z interrupted in-flight Helm upgrade

## Resolution

- Clear the Stalled state and let Flux retry the upgrade. First try: `flux reconcile hr victoria-metrics-k8s-stack -n observability --force`. If that does not clear Stalled (likely, since RetriesExceeded is terminal), escalate to: `flux reconcile hr victoria-metrics-k8s-stack -n observability --reset` — this clears the failure counters and triggers a fresh upgrade attempt. The helm release storage is NOT wedged (revision 4 is still 'deployed'), so `helm uninstall` should NOT be needed. The underlying workloads are all healthy; the upgrade should succeed since the original failure was a transient 'context canceled' interruption, not a chart or capacity problem. Consider adding `upgrade.remediation.retries: 3` to the HelmRelease spec to prevent a single transient interruption from causing a terminal failure in the future. (reversible=true)

## Unresolved

- What specific content changed in ConfigMap/observability/vm-common-helm-values that triggered the reconciliation interrupting the upgrade — the HelmRelease status references it but what_changed did not surface a diff for this ConfigMap (it is likely managed by a different Kustomization or applied out-of-band); however, this does not affect the fix since the interrupted upgrade just needs to be retried

## Citations

[1] ConfigMap/observability/vm-common-helm-values change at ~15:14:17Z interrupted in-flight Helm upgrade

