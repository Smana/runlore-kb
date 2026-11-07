---
type: Incident
title: 'MetricsServerUnavailable fires on a typo: VMRule runlore-kb-seed queries deployment="metrics-servr" (missing ''e''),…'
description: 'The alerting rule in VMRule observability/runlore-kb-seed (group name metrics-server-availability) has a typo in its expression: deployment="metrics-servr" — missing the final ''e''. There is no Deployment named metrics-servr, so kube_deployment_status_replicas_available{namespace="kube-system",deployment="metrics-servr"} matches zero series. absent(... > 0) on an empty result returns vector(1), making the alert condition permanently true. After the 5m for: hold-down, the alert fires continuously regardless of the real metrics-server health. The actual Deployment kube-system/metrics-server is healthy: 2/2 replicas available, both pods Running 1/1, status Available=True since 2026-11-07T09:22:19Z — hours before the alert. The VMRule was applied at 2026-11-07T15:47:38Z; the alert started at 15:52:00Z, ~4-5 minutes later, consistent with the 5m hold-down.'
resource: observability/runlore-kb-seed
tags:
    - runlore
    - incident
    - vmrule
    - observability
timestamp: "2026-11-07T19:06:03Z"
fingerprint: 5707f78ef4a6e5d66ece950e4453686cf5b443ebd646955ff998d365e03b092d
confidence: 0.92
provenance:
    - VMRule observability/runlore-kb-seed applied at 2026-11-07T15:47:38Z with typo'd deployment label metrics-servr
origin: runlore
resolution:
    fix_pr: https://github.com/Smana/cloud-native-ref/pull/2754
    merge_sha: f3737044ae3961f7ed4f5e2978210a665e00fc23
    verdict: resolved
    verified_at: "2026-11-07T19:05:11Z"
---

## Decision

- **why keep:** The alerting rule in VMRule observability/runlore-kb-seed (group name metrics-server-availability) has a typo in its expression: deployment="metrics-servr" — missing the final 'e'. There is no Deployment named metrics-servr, so kube_deployment_status_replicas_available{namespace="kube-system",deployment="metrics-servr"} matches zero series. absent(... > 0) on an empty result returns vector(1), making the alert condition permanently true. After the 5m for: hold-down, the alert fires continuously regardless of the real metrics-server health. The actual Deployment kube-system/metrics-server is healthy: 2/2 replicas available, both pods Running 1/1, status Available=True since 2026-11-07T09:22:19Z — hours before the alert. The VMRule was applied at 2026-11-07T15:47:38Z; the alert started at 15:52:00Z, ~4-5 minutes later, consistent with the 5m hold-down.
- **confidence:** 92%
- **provenance:** VMRule observability/runlore-kb-seed applied at 2026-11-07T15:47:38Z with typo'd deployment label metrics-servr

## Symptom

MetricsServerUnavailable fires on a typo: VMRule runlore-kb-seed queries deployment="metrics-servr" (missing 'e'), which matches no series — metrics-server itself is healthy (2/2)

Affected resource: VMRule observability/runlore-kb-seed

## Investigate

- alert_rule(MetricsServerUnavailable): expr = absent(kube_deployment_status_replicas_available{namespace="kube-system",deployment="metrics-servr"} > 0), for=5m, state=firing, file=/etc/vmalert/rules-out/rules-src-0/rules.yaml — note the typo 'metrics-servr'
- query_metrics(kube_deployment_status_replicas_available{namespace="kube-system",deployment="metrics-servr"}) → 'no series matched' — the typo'd deployment name matches nothing
- query_metrics(kube_deployment_status_replicas_available{namespace="kube-system",deployment="metrics-server"}) → = 2 — the CORRECT deployment name shows 2 available replicas right now
- query_metrics_range(kube_deployment_status_replicas_available{namespace="kube-system",deployment="metrics-server"}) over 20m → first=2 last=2 min=2 max=2, flat at 2 throughout — never dropped to zero
- resource_spec(Deployment kube-system/metrics-server): status.availableReplicas=2, readyReplicas=2, replicas=2; condition Available=True reason=MinimumReplicasAvailable since 2026-11-07T09:22:19Z; condition Progressing=True reason=NewReplicaSetAvailable
- pod_status(kube-system, k8s-app=metrics-server): both pods metrics-server-5b6965cd84-8xh62 and metrics-server-5b6965cd84-z6x9l Running ready=1/1 age=6h — no restarts, no failures
- resource_spec(VMRule observability/runlore-kb-seed): spec.groups[0].name=metrics-server-availability, rules[0].alert=MetricsServerUnavailable, expr=absent(kube_deployment_status_replicas_available{namespace="kube-system",deployment="metrics-servr"} > 0); status condition Applied=True lastTransitionTime=2026-11-07T15:47:38Z — the typo is in this VMRule's source

## Cause

1. **The alerting rule in VMRule observability/runlore-kb-seed (group name metrics-server-availability) has a typo in its expression: deployment="metrics-servr" — missing the final 'e'. There is no Deployment named metrics-servr, so kube_deployment_status_replicas_available{namespace="kube-system",deployment="metrics-servr"} matches zero series. absent(... > 0) on an empty result returns vector(1), making the alert condition permanently true. After the 5m for: hold-down, the alert fires continuously regardless of the real metrics-server health. The actual Deployment kube-system/metrics-server is healthy: 2/2 replicas available, both pods Running 1/1, status Available=True since 2026-11-07T09:22:19Z — hours before the alert. The VMRule was applied at 2026-11-07T15:47:38Z; the alert started at 15:52:00Z, ~4-5 minutes later, consistent with the 5m hold-down.** (92%) — change: VMRule observability/runlore-kb-seed applied at 2026-11-07T15:47:38Z with typo'd deployment label metrics-servr

## Resolution

- **Applied:** [#2754](https://github.com/Smana/cloud-native-ref/pull/2754)
- **Diff:** 1 file: observability/aws-0/vmrule-runlore-kb-seed.yaml (+2 −5)
- **Soak evidence:** firing at merge: false, firing now: false.

<!-- runlore-resolution:2e77314c -->

### Suggested by the investigation

- Fix the typo in VMRule observability/runlore-kb-seed: change deployment="metrics-servr" to deployment="metrics-server" in the MetricsServerUnavailable alert expression. The rule lives in the Git source for that VMRule (the runlore-kb-seed VMRule). After the fix is applied and VMAlert reloads (~1-2 min), the alert will clear because the correct series (kube_deployment_status_replicas_available{deployment="metrics-server"}=2) makes absent() return empty, so the alert condition becomes false. (reversible=true)

## Citations

[1] VMRule observability/runlore-kb-seed applied at 2026-11-07T15:47:38Z with typo'd deployment label metrics-servr

