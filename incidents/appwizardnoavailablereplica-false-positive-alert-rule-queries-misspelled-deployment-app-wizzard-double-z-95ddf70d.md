---
type: Incident
title: 'AppWizardNoAvailableReplica — false positive: alert rule queries misspelled deployment "app-wizzard" (double z),…'
description: 'The alert rule AppWizardNoAvailableReplica in the vmalert rules file (/etc/vmalert/rules-out/rules-src-0/rules.yaml) has a typo in its PromQL expression: it filters on deployment="app-wizzard" (TWO z''s) instead of the correct deployment="app-wizard" (ONE z). Because no Deployment named "app-wizzard" exists, the selector kube_deployment_status_replicas_available{namespace="apps", deployment="app-wizzard"} matches no series. The absent(... > 0) wrapper then always returns 1 (vector present = alert firing), so the alert fires permanently regardless of the actual deployment health. The real Deployment apps/app-wizard is fully healthy with availableReplicas=1, readyReplicas=1, and its pod app-wizard-979c7dcfd-v4nj4 is Running ready=6/6.'
resource: observability/AppWizardNoAvailableReplica
tags:
    - runlore
    - incident
    - prometheusrule
    - observability
timestamp: "2026-10-28T22:58:37Z"
fingerprint: 95ddf70dc8e812d9d2fec5633b9df696a9b9322e4705196e832612d7c413f468
confidence: 0.85
provenance:
    - Alert rule in /etc/vmalert/rules-out/rules-src-0/rules.yaml, group=app-wizard, alertname=AppWizardNoAvailableReplica
---

## Decision

- **why keep:** The alert rule AppWizardNoAvailableReplica in the vmalert rules file (/etc/vmalert/rules-out/rules-src-0/rules.yaml) has a typo in its PromQL expression: it filters on deployment="app-wizzard" (TWO z's) instead of the correct deployment="app-wizard" (ONE z). Because no Deployment named "app-wizzard" exists, the selector kube_deployment_status_replicas_available{namespace="apps", deployment="app-wizzard"} matches no series. The absent(... > 0) wrapper then always returns 1 (vector present = alert firing), so the alert fires permanently regardless of the actual deployment health. The real Deployment apps/app-wizard is fully healthy with availableReplicas=1, readyReplicas=1, and its pod app-wizard-979c7dcfd-v4nj4 is Running ready=6/6.
- **confidence:** 85%
- **provenance:** Alert rule in /etc/vmalert/rules-out/rules-src-0/rules.yaml, group=app-wizard, alertname=AppWizardNoAvailableReplica

## Symptom

AppWizardNoAvailableReplica — false positive: alert rule queries misspelled deployment "app-wizzard" (double z), actual deployment "app-wizard" is healthy

Affected resource: PrometheusRule observability/AppWizardNoAvailableReplica

## Investigate

- alert_rule returned: expr: absent(kube_deployment_status_replicas_available{namespace="apps", deployment="app-wizzard"} > 0) — note the double-z in 'app-wizzard'
- query_metrics for the misspelled name: kube_deployment_status_replicas_available{namespace="apps", deployment="app-wizzard"} → 'no series matched'
- query_metrics for the correct name: kube_deployment_status_replicas_available{namespace="apps", deployment="app-wizard"} = 1 (one available replica)
- query_metrics_range for the correct name over 30min: first=1 last=1 min=1 max=1 trend=1>1>1>... — flat at 1 the entire window, never dropped to 0
- resource_spec Deployment apps/app-wizard: status.availableReplicas=1, status.readyReplicas=1, condition Available=True (lastTransitionTime 2026-10-27T07:37:46Z), condition Progressing=True
- pod_status: app-wizard-979c7dcfd-v4nj4 Running ready=6/6 age=4m — all 6 containers healthy
- list_resources Deployment in apps: only app-wizard, image-gallery, podinfo, and two xplane-* deployments exist — no deployment named 'app-wizzard' exists anywhere in the namespace
- kube_events in apps (last 30m): no Warning events at all — no pod failures, no scheduling issues, no probe failures

## Cause

1. **The alert rule AppWizardNoAvailableReplica in the vmalert rules file (/etc/vmalert/rules-out/rules-src-0/rules.yaml) has a typo in its PromQL expression: it filters on deployment="app-wizzard" (TWO z's) instead of the correct deployment="app-wizard" (ONE z). Because no Deployment named "app-wizzard" exists, the selector kube_deployment_status_replicas_available{namespace="apps", deployment="app-wizzard"} matches no series. The absent(... > 0) wrapper then always returns 1 (vector present = alert firing), so the alert fires permanently regardless of the actual deployment health. The real Deployment apps/app-wizard is fully healthy with availableReplicas=1, readyReplicas=1, and its pod app-wizard-979c7dcfd-v4nj4 is Running ready=6/6.** (85%) — change: Alert rule in /etc/vmalert/rules-out/rules-src-0/rules.yaml, group=app-wizard, alertname=AppWizardNoAvailableReplica

## Resolution

- Fix the typo in the alerting rule expression: change deployment="app-wizzard" to deployment="app-wizard" in the PrometheusRule/VMRule that defines the AppWizardNoAvailableReplica alert. The rule is loaded by the observability-victoria-metrics-k8s-stack Kustomization (sourceRef: ExternalArtifact flux-system/observability-artifact) and served at /etc/vmalert/rules-out/rules-src-0/rules.yaml. After fixing, the absent() expression will correctly evaluate against the real app-wizard deployment metric and the alert will stop firing. (reversible=false)

## Citations

[1] Alert rule in /etc/vmalert/rules-out/rules-src-0/rules.yaml, group=app-wizard, alertname=AppWizardNoAvailableReplica

