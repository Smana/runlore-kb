---
type: Incident
title: '[PRE-EXISTING, occurrence #4] HelmRelease victoria-metrics-k8s-stack UpgradeFailed: Grafana CrashLoopBackOff due to…'
description: 'The HelmRelease observability/victoria-metrics-k8s-stack (chart victoria-metrics-k8s-stack@0.97.1) configures Grafana with GF_PLUGINS_PREINSTALL_SYNC=victoriametrics-metrics-datasource@1.0.1,victoriametrics-logs-datasource@0.34.0,marcusolsson-dynamictext-panel. Version 0.34.0 of the victoriametrics-logs-datasource plugin does not exist in the Grafana.com catalog ([plugin.versionNotFound] error). Grafana''s plugin.backgroundinstaller module fails on startup, which cascades to all dependent modules (provisioning, api.HTTPServer, sqlstore, etc.) failing, causing Grafana to exit with code 1 and enter CrashLoopBackOff. The Helm upgrade then fails because the Deployment is in ''Failed'' status (stalled resources). The pod victoria-metrics-k8s-stack-grafana-7c444d778c-hn24h has 42 restarts and is in CrashLoopBackOff with back-off 5m0s. This is the SAME fault as occurrence #3 (58m ago) — it has NOT been remediated.'
resource: observability/victoria-metrics-k8s-stack-grafana
alert_resource: observability/victoria-metrics-k8s-stack
tags:
    - runlore
    - incident
    - deployment
    - observability
timestamp: "2026-11-08T14:42:18Z"
fingerprint: bcaf743f85176cdb8c87c34954d38596f9c22f28897cc7b80ea06104bcea988e
confidence: 0.92
provenance:
    - HelmRelease victoria-metrics-k8s-stack chart 0.97.1 first deployed 2026-11-07T09:17:17Z
origin: runlore
---

## Decision

- **why keep:** The HelmRelease observability/victoria-metrics-k8s-stack (chart victoria-metrics-k8s-stack@0.97.1) configures Grafana with GF_PLUGINS_PREINSTALL_SYNC=victoriametrics-metrics-datasource@1.0.1,victoriametrics-logs-datasource@0.34.0,marcusolsson-dynamictext-panel. Version 0.34.0 of the victoriametrics-logs-datasource plugin does not exist in the Grafana.com catalog ([plugin.versionNotFound] error). Grafana's plugin.backgroundinstaller module fails on startup, which cascades to all dependent modules (provisioning, api.HTTPServer, sqlstore, etc.) failing, causing Grafana to exit with code 1 and enter CrashLoopBackOff. The Helm upgrade then fails because the Deployment is in 'Failed' status (stalled resources). The pod victoria-metrics-k8s-stack-grafana-7c444d778c-hn24h has 42 restarts and is in CrashLoopBackOff with back-off 5m0s. This is the SAME fault as occurrence #3 (58m ago) — it has NOT been remediated.
- **confidence:** 92%
- **provenance:** HelmRelease victoria-metrics-k8s-stack chart 0.97.1 first deployed 2026-11-07T09:17:17Z

## Symptom

[PRE-EXISTING, occurrence #4] HelmRelease victoria-metrics-k8s-stack UpgradeFailed: Grafana CrashLoopBackOff due to non-existent plugin version victoriametrics-logs-datasource@0.34.0

Affected resource: Deployment observability/victoria-metrics-k8s-stack-grafana

## Investigate

- query_logs on the crashing pod (7c444d778c): 'failed to install plugin victoriametrics-logs-datasource@0.34.0: [plugin.versionNotFound] victoriametrics-logs-datasource v0.34.0 was not returned by the Grafana.com catalog. The version may not exist, or the configured Grafana.com proxy token may lack the required scopes.' (x11 occurrences, first 13:42:54Z → last 14:34:50Z)
- query_logs: 'Config overridden from Environment variable' var='GF_PLUGINS_PREINSTALL_SYNC=victoriametrics-metrics-datasource@1.0.1,victoriametrics-logs-datasource@0.34.0,marcusolsson-dynamictext-panel' — confirms the plugin version is injected via env var from the Helm chart config
- pod_status: victoria-metrics-k8s-stack-grafana-7c444d778c-hn24h Running ready=2/3 restarts=42 CrashLoopBackOff: back-off 5m0s restarting failed container=grafana [last termination: Error (exit 1)]
- gitops_resource_status HelmRelease: Ready=False reason=UpgradeFailed, message 'failed early due to stalled resources: [Deployment/observability/victoria-metrics-k8s-stack-grafana status: Failed]', 87 failures
- query_metrics_range kube_pod_container_status_restarts_total: grafana container restarts ramped from 8@11:40 to 42@14:36, steady ~3min crash cycle, confirming continuous CrashLoopBackOff
- workload_ownership: Pod → ReplicaSet → Deployment/victoria-metrics-k8s-stack-grafana → managed by HelmRelease observability/victoria-metrics-k8s-stack (flux), drift: none detected
- pod b2qvd (old ReplicaSet 689d7d8b8d, ready=3/3) runs fine — it predates the chart upgrade and does not have the bad plugin version configured

## Cause

1. **The HelmRelease observability/victoria-metrics-k8s-stack (chart victoria-metrics-k8s-stack@0.97.1) configures Grafana with GF_PLUGINS_PREINSTALL_SYNC=victoriametrics-metrics-datasource@1.0.1,victoriametrics-logs-datasource@0.34.0,marcusolsson-dynamictext-panel. Version 0.34.0 of the victoriametrics-logs-datasource plugin does not exist in the Grafana.com catalog ([plugin.versionNotFound] error). Grafana's plugin.backgroundinstaller module fails on startup, which cascades to all dependent modules (provisioning, api.HTTPServer, sqlstore, etc.) failing, causing Grafana to exit with code 1 and enter CrashLoopBackOff. The Helm upgrade then fails because the Deployment is in 'Failed' status (stalled resources). The pod victoria-metrics-k8s-stack-grafana-7c444d778c-hn24h has 42 restarts and is in CrashLoopBackOff with back-off 5m0s. This is the SAME fault as occurrence #3 (58m ago) — it has NOT been remediated.** (92%) — change: HelmRelease victoria-metrics-k8s-stack chart 0.97.1 first deployed 2026-11-07T09:17:17Z

## Resolution

- Update the Grafana plugin configuration to use a valid version of victoriametrics-logs-datasource. The plugin version 0.34.0 does not exist in the Grafana.com catalog — either pin a version that does exist (check https://grafana.com/grafana/plugins/victoriametrics-logs-datasource/ for available versions) or remove the version pin entirely to let Grafana install the latest. The configuration is set via GF_PLUGINS_PREINSTALL_SYNC, likely in the HelmRelease values or the vm-common-helm-values ConfigMap (RBAC prevented direct reading of these ConfigMaps). After fixing, the HelmRelease should reconcile successfully as the Grafana pod will start normally. (reversible=true)

## Unresolved

- The exact available version of victoriametrics-logs-datasource that should replace 0.34.0 could not be determined from cluster-side tools — a human should check the Grafana.com plugin catalog at https://grafana.com/grafana/plugins/victoriametrics-logs-datasource/ for the correct version.

## Citations

[1] HelmRelease victoria-metrics-k8s-stack chart 0.97.1 first deployed 2026-11-07T09:17:17Z

