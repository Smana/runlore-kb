---
type: Incident
title: PodinfoNoAvailableReplica alert is a false positive — rule expression has a typo (deployment="podnfo" instead of…
description: 'The alerting rule PodinfoNoAvailableReplica in /etc/vmalert/rules-out/rules-src-0/rules.yaml has a typo in its PromQL expression: the deployment label is "podnfo" (missing the letter ''i'') instead of "podinfo". Because no Deployment named "podnfo" exists, the metric kube_deployment_status_replicas_available{namespace="apps", deployment="podnfo"} returns no series. Wrapping a non-existent series in absent() always evaluates to 1, so the alert fires permanently regardless of the actual health of the podinfo Deployment. The real podinfo Deployment has 1 available replica (Running, ready=1/1, age=9h) and has been steady at 1 for the entire incident window.'
resource: observability/PodinfoNoAvailableReplica
tags:
    - runlore
    - incident
    - alertingrule
    - observability
timestamp: "2026-10-28T17:29:12Z"
fingerprint: a386335811c34d60af97ce99c592701b4945e3cfd6c08f0ec56dce00c1369a31
confidence: 0.92
---

## Decision

- **why keep:** The alerting rule PodinfoNoAvailableReplica in /etc/vmalert/rules-out/rules-src-0/rules.yaml has a typo in its PromQL expression: the deployment label is "podnfo" (missing the letter 'i') instead of "podinfo". Because no Deployment named "podnfo" exists, the metric kube_deployment_status_replicas_available{namespace="apps", deployment="podnfo"} returns no series. Wrapping a non-existent series in absent() always evaluates to 1, so the alert fires permanently regardless of the actual health of the podinfo Deployment. The real podinfo Deployment has 1 available replica (Running, ready=1/1, age=9h) and has been steady at 1 for the entire incident window.
- **confidence:** 92%

## Symptom

PodinfoNoAvailableReplica alert is a false positive — rule expression has a typo (deployment="podnfo" instead of "podinfo")

Affected resource: AlertingRule observability/PodinfoNoAvailableReplica

## Investigate

- alert_rule returned: expr=absent(kube_deployment_status_replicas_available{namespace="apps", deployment="podnfo"} > 0) — note the label value is 'podnfo', not 'podinfo'
- query_metrics for deployment="podnfo": 'no series matched — the metric name or labels may not exist'
- query_metrics for deployment="podinfo": kube_deployment_status_replicas_available{namespace="apps",deployment="podinfo"} = 1
- query_metrics for the full alert expression absent(kube_deployment_status_replicas_available{namespace="apps", deployment="podnfo"} > 0): {} = 1 — the alert expression evaluates to 1 (firing) because absent() of a non-existent series always returns 1
- query_metrics_range over 120m: kube_deployment_status_replicas_available{namespace="apps",deployment="podinfo"} first=1 last=1 min=1 max=1 trend=1>1>1>1>1>1>1>1>1>1>1>1 — flat at 1 available replica across the entire window
- pod_status (apps namespace): podinfo-644b6469b9-4s57d Running ready=1/1 age=9h — the pod is healthy and has been running for 9 hours
- kube_events (apps namespace, 120m): no Warning events for podinfo — only unrelated cnpg-cluster liveness/readiness probe failures for xplane-image-gallery-cnpg-cluster-1

## Cause

1. **The alerting rule PodinfoNoAvailableReplica in /etc/vmalert/rules-out/rules-src-0/rules.yaml has a typo in its PromQL expression: the deployment label is "podnfo" (missing the letter 'i') instead of "podinfo". Because no Deployment named "podnfo" exists, the metric kube_deployment_status_replicas_available{namespace="apps", deployment="podnfo"} returns no series. Wrapping a non-existent series in absent() always evaluates to 1, so the alert fires permanently regardless of the actual health of the podinfo Deployment. The real podinfo Deployment has 1 available replica (Running, ready=1/1, age=9h) and has been steady at 1 for the entire incident window.** (92%)

## Resolution

- Fix the typo in the alert rule expression: change deployment="podnfo" to deployment="podinfo" in the PodinfoNoAvailableReplica rule in /etc/vmalert/rules-out/rules-src-0/rules.yaml (sourced from Git — find and correct it in the repository that renders this rules file, likely under the observability GitOps path). Once corrected, absent() will evaluate against the real podinfo series, which currently returns 1 available replica, and the alert will stop firing. (reversible=true)

## Unresolved

- Could not identify the exact Git commit that introduced the typo, because what_changed on the observability namespace returned an unrelated Kustomization sync from 2026-08-24 (with a diff error). The GitOps object that renders /etc/vmalert/rules-out/rules-src-0/rules.yaml was not found in the observability namespace's recent history — the rule source may be managed by a GitOps object in a different namespace or from a config not surfaced by what_changed.

