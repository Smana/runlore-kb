---
type: Incident
title: '[PRE-EXISTING] HelmRelease victoria-metrics-k8s-stack UpgradeFailed: Grafana CrashLoopBackOff due to non-existent…'
description: Grafana pod victoria-metrics-k8s-stack-grafana-7c444d778c-hn24h is in CrashLoopBackOff (18 restarts, exit 1) because the background plugin installer fails to download plugin victoriametrics-logs-datasource@0.34.0 — version v0.34.0 is not found in the Grafana.com catalog ([plugin.versionNotFound] 'victoriametrics-logs-datasource v0.34.0 was not returned by the Grafana.com catalog'). The plugin version is pinned in the HelmRelease's values (sourced from ConfigMap observability/vm-common-helm-values, referenced via spec.valuesFrom). When the background installer module fails, all dependent Grafana modules fail cascadingly, causing the grafana container to exit with error. This leaves Deployment/victoria-metrics-k8s-stack-grafana in Failed status, which causes the Helm upgrade to stall and the HelmRelease to fail with UpgradeFailed.
resource: observability/victoria-metrics-k8s-stack
tags:
    - runlore
    - incident
    - helmrelease
    - observability
timestamp: "2026-11-08T12:38:15Z"
fingerprint: 25bf9081c484d74ff6626ee36139c41e5d8815862ade3cd2e5011fa7bc43bd3c
confidence: 0.88
provenance:
    - observability/vm-common-helm-values ConfigMap (RBAC-forbidden to read directly) referenced by HelmRelease spec.valuesFrom; the same configDigest (sha256:6101bc7c…) appears across all 30 failed upgrade attempts, confirming the config has not changed since the first failure
origin: runlore
---

## Decision

- **why keep:** Grafana pod victoria-metrics-k8s-stack-grafana-7c444d778c-hn24h is in CrashLoopBackOff (18 restarts, exit 1) because the background plugin installer fails to download plugin victoriametrics-logs-datasource@0.34.0 — version v0.34.0 is not found in the Grafana.com catalog ([plugin.versionNotFound] 'victoriametrics-logs-datasource v0.34.0 was not returned by the Grafana.com catalog'). The plugin version is pinned in the HelmRelease's values (sourced from ConfigMap observability/vm-common-helm-values, referenced via spec.valuesFrom). When the background installer module fails, all dependent Grafana modules fail cascadingly, causing the grafana container to exit with error. This leaves Deployment/victoria-metrics-k8s-stack-grafana in Failed status, which causes the Helm upgrade to stall and the HelmRelease to fail with UpgradeFailed.
- **confidence:** 88%
- **provenance:** observability/vm-common-helm-values ConfigMap (RBAC-forbidden to read directly) referenced by HelmRelease spec.valuesFrom; the same configDigest (sha256:6101bc7c…) appears across all 30 failed upgrade attempts, confirming the config has not changed since the first failure

## Symptom

[PRE-EXISTING] HelmRelease victoria-metrics-k8s-stack UpgradeFailed: Grafana CrashLoopBackOff due to non-existent plugin version victoriametrics-logs-datasource@0.34.0

Affected resource: HelmRelease observability/victoria-metrics-k8s-stack

## Investigate

- pod_status: grafana pod victoria-metrics-k8s-stack-grafana-7c444d778c-hn24h is CrashLoopBackOff with 18 restarts, last-exit=2026-11-08T12:31:14Z, exit code 1, ready=2/3
- query_logs (grafana container, previous=true): 'level=error msg="Failed to install plugins" error="failed to install plugin victoriametrics-logs-datasource@0.34.0: [plugin.versionNotFound] victoriametrics-logs-datasource v0.34.0 was not returned by the Grafana.com catalog"' (x17 occurrences from 11:23:44Z to 12:31:14Z)
- query_logs: 'Installing plugins' plugins="[{ID:victoriametrics-metrics-datasource Version:1.0.1 URL:} {ID:victoriametrics-logs-datasource Version:0.34.0 URL:} {ID:marcusolsson-dynamictext-panel Version: URL:}]' — confirms v0.34.0 is explicitly pinned
- gitops_resource_status: HelmRelease Ready=False reason=UpgradeFailed, message='failed early due to stalled resources: [Deployment/observability/victoria-metrics-k8s-stack-grafana status: Failed]', status.failures=30
- controller_logs: helm-controller repeatedly logs 'release is in a failed state' then 'running upgrade action with timeout of 30m0s' every ~2m, each attempt failing
- query_logs: marcusolsson-dynamictext-panel installed successfully ('Plugin successfully installed') while victoriametrics-logs-datasource@0.34.0 fails — the other plugins are fine, isolating the fault to v0.34.0

## Cause

1. **Grafana pod victoria-metrics-k8s-stack-grafana-7c444d778c-hn24h is in CrashLoopBackOff (18 restarts, exit 1) because the background plugin installer fails to download plugin victoriametrics-logs-datasource@0.34.0 — version v0.34.0 is not found in the Grafana.com catalog ([plugin.versionNotFound] 'victoriametrics-logs-datasource v0.34.0 was not returned by the Grafana.com catalog'). The plugin version is pinned in the HelmRelease's values (sourced from ConfigMap observability/vm-common-helm-values, referenced via spec.valuesFrom). When the background installer module fails, all dependent Grafana modules fail cascadingly, causing the grafana container to exit with error. This leaves Deployment/victoria-metrics-k8s-stack-grafana in Failed status, which causes the Helm upgrade to stall and the HelmRelease to fail with UpgradeFailed.** (88%) — change: observability/vm-common-helm-values ConfigMap (RBAC-forbidden to read directly) referenced by HelmRelease spec.valuesFrom; the same configDigest (sha256:6101bc7c…) appears across all 30 failed upgrade attempts, confirming the config has not changed since the first failure

## Resolution

- In the Git-sourced Helm values (ConfigMap observability/vm-common-helm-values, key values.yaml, under grafana.plugins), change the pinned version for victoriametrics-logs-datasource from 0.34.0 to a version that exists in the Grafana.com catalog, or remove the version pin to let Grafana install the latest. Then force-reconcile: flux reconcile hr victoria-metrics-k8s-stack -n observability --force (reversible=true)

## Unresolved

- The exact available version of victoriametrics-logs-datasource to pin instead of 0.34.0 requires checking the Grafana.com plugin catalog externally — not resolvable with cluster-internal tooling

## Citations

[1] observability/vm-common-helm-values ConfigMap (RBAC-forbidden to read directly) referenced by HelmRelease spec.valuesFrom; the same configDigest (sha256:6101bc7c…) appears across all 30 failed upgrade attempts, confirming the config has not changed since the first failure

