---
type: Incident
title: 'Grafana CrashLoopBackOff: victoriametrics-logs-datasource@0.34.0 plugin version not found, blocking HelmRelease upgrade'
description: A GitOps change to the victoria-metrics-k8s-stack HelmRelease (revision 35b0d86b synced at 2026-11-08T11:23:02Z) changed Grafana's GF_PLUGINS_PREINSTALL_SYNC config to pin victoriametrics-logs-datasource@0.34.0, a version that does not exist in the Grafana.com catalog ([plugin.versionNotFound]). Grafana 13.1.4 treats plugin preinstall failure as a fatal startup error (the plugin.backgroundinstaller module fails, cascading to all dependent modules), so the new grafana container exits with code 1 and enters CrashLoopBackOff. The old healthy pod (689d7d8b8d-b2qvd) ran version 0.33.1 successfully. The Deployment never reaches Ready, so Helm's --wait times out, the HelmRelease goes UpgradeFailed, and the owning Kustomization reports HealthCheckFailed. The fix is to revert or correct the plugin version in the Helm values (back to 0.33.1 or a valid published version), then force-reconcile the HelmRelease.
resource: observability/victoria-metrics-k8s-stack-grafana
alert_resource: flux-system/observability-victoria-metrics-k8s-stack
tags:
    - runlore
    - incident
    - deployment
    - observability
timestamp: "2026-11-08T11:38:40Z"
fingerprint: 0cf059557eaa194583ad2830b92693b7b1a6210e1309e6b7221c0ecc9c317e66
confidence: 0.85
provenance:
    - flux Kustomization/observability-victoria-metrics-k8s-stack synced to revision 35b0d86b74ccba3317b8cce827d351d8f8b96216a9e01f8d84968cd7adfc94a9 at 2026-11-08T11:23:02Z
origin: runlore
---

## Decision

- **why keep:** A GitOps change to the victoria-metrics-k8s-stack HelmRelease (revision 35b0d86b synced at 2026-11-08T11:23:02Z) changed Grafana's GF_PLUGINS_PREINSTALL_SYNC config to pin victoriametrics-logs-datasource@0.34.0, a version that does not exist in the Grafana.com catalog ([plugin.versionNotFound]). Grafana 13.1.4 treats plugin preinstall failure as a fatal startup error (the plugin.backgroundinstaller module fails, cascading to all dependent modules), so the new grafana container exits with code 1 and enters CrashLoopBackOff. The old healthy pod (689d7d8b8d-b2qvd) ran version 0.33.1 successfully. The Deployment never reaches Ready, so Helm's --wait times out, the HelmRelease goes UpgradeFailed, and the owning Kustomization reports HealthCheckFailed. The fix is to revert or correct the plugin version in the Helm values (back to 0.33.1 or a valid published version), then force-reconcile the HelmRelease.
- **confidence:** 85%
- **provenance:** flux Kustomization/observability-victoria-metrics-k8s-stack synced to revision 35b0d86b74ccba3317b8cce827d351d8f8b96216a9e01f8d84968cd7adfc94a9 at 2026-11-08T11:23:02Z

## Symptom

Grafana CrashLoopBackOff: victoriametrics-logs-datasource@0.34.0 plugin version not found, blocking HelmRelease upgrade

Affected resource: Deployment observability/victoria-metrics-k8s-stack-grafana

## Investigate

- query_logs: New pod (7c444d778c-hn24h) crash error: 'Error: ✗ invalid service state: Failed, expected: Terminated, failure: starting module plugin.backgroundinstaller: invalid service state: Failed, expected: Running, failure: failed to install plugin victoriametrics-logs-datasource@0.34.0: [plugin.versionNotFound] victoriametrics-logs-datasource v0.34.0 was not returned by the Grafana.com catalog.' (x6 crashes from 11:23:44 to 11:34:31)
- query_logs: Env var on new pod confirms the pinned version: 'GF_PLUGINS_PREINSTALL_SYNC=victoriametrics-metrics-datasource@1.0.1,victoriametrics-logs-datasource@0.34.0,marcusolsson-dynamictext-panel'
- query_logs: Old healthy pod (689d7d8b8d-b2qvd) log lines from 2026-11-07 show the working version: 'pluginId=victoriametrics-logs-datasource pluginVersion=0.33.1' (multiple successful Plugin Request Completed entries)
- pod_status: 'victoria-metrics-k8s-stack-grafana-7c444d778c-hn24h Running ready=2/3 restarts=6 age=11m last-exit=2026-11-08T11:29:26Z ... grafana: CrashLoopBackOff: back-off 5m0s restarting failed container' — while old pod 689d7d8b8d-b2qvd is 'Running ready=3/3 age=1d'
- gitops_resource_status HelmRelease: Ready=False (UpgradeFailed), message='Helm upgrade failed for release observability/victoria-metrics-k8s-stack with chart victoria-metrics-k8s-stack@0.97.1: failed early due to stalled resources: [Deployment/observability/victoria-metrics-k8s-stack-grafana status: Failed]'
- controller_logs helm-controller: 'release out-of-sync with desired state: release config values changed' at 11:23:02, followed by 'running upgrade action' and then 'release is in a failed state' at 11:33:09
- what_changed: Kustomization/observability-victoria-metrics-k8s-stack synced to new revision 35b0d86b at 2026-11-08T11:33:02Z
- resource_spec Deployment: status shows ProgressDeadlineExceeded, replicas=2 (old + new), updatedReplicas=1, readyReplicas=1, unavailableReplicas=1
- kube_events: 'BackOff (x19): Back-off restarting failed container grafana' and 'Unhealthy (x14): Readiness probe failed: dial tcp 100.64.19.120:3000: connect: connection refused' on the new pod

## Cause

1. **A GitOps change to the victoria-metrics-k8s-stack HelmRelease (revision 35b0d86b synced at 2026-11-08T11:23:02Z) changed Grafana's GF_PLUGINS_PREINSTALL_SYNC config to pin victoriametrics-logs-datasource@0.34.0, a version that does not exist in the Grafana.com catalog ([plugin.versionNotFound]). Grafana 13.1.4 treats plugin preinstall failure as a fatal startup error (the plugin.backgroundinstaller module fails, cascading to all dependent modules), so the new grafana container exits with code 1 and enters CrashLoopBackOff. The old healthy pod (689d7d8b8d-b2qvd) ran version 0.33.1 successfully. The Deployment never reaches Ready, so Helm's --wait times out, the HelmRelease goes UpgradeFailed, and the owning Kustomization reports HealthCheckFailed. The fix is to revert or correct the plugin version in the Helm values (back to 0.33.1 or a valid published version), then force-reconcile the HelmRelease.** (85%) — change: flux Kustomization/observability-victoria-metrics-k8s-stack synced to revision 35b0d86b74ccba3317b8cce827d351d8f8b96216a9e01f8d84968cd7adfc94a9 at 2026-11-08T11:23:02Z

## Resolution

- In the Git source (observability/base/victoria-metrics-k8s-stack HelmRelease values or vm-common-helm-values ConfigMap), revert the victoriametrics-logs-datasource plugin version from 0.34.0 back to 0.33.1 (or verify which versions are actually published on the Grafana.com catalog and use a valid one). Then force-reconcile the HelmRelease: `flux reconcile hr victoria-metrics-k8s-stack -n observability --force`. The old pod (689d7d8b8d-b2qvd) is still serving with 0.33.1 and ready=3/3, so impact is limited to the failed rollout until the fix lands. (reversible=true)

## Citations

[1] flux Kustomization/observability-victoria-metrics-k8s-stack synced to revision 35b0d86b74ccba3317b8cce827d351d8f8b96216a9e01f8d84968cd7adfc94a9 at 2026-11-08T11:23:02Z

