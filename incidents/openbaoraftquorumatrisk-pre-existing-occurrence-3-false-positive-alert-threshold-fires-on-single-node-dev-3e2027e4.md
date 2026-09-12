---
type: Incident
title: 'OpenBaoRaftQuorumAtRisk (pre-existing, occurrence #3) — false positive: alert threshold fires on single-node dev…'
description: 'PRE-EXISTING FALSE POSITIVE (unchanged from occurrence #2): The alert rule `vault_autopilot_failure_tolerance{job="openbao"} < 1` fires on a single-node dev OpenBao cluster where failure_tolerance=0 is the mathematically correct and expected state. For a 1-node raft cluster, failure tolerance = (N-1)/2 = (1-1)/2 = 0. The rule is designed for multi-node HA clusters (3+ nodes, where failure_tolerance < 1 would indicate a real quorum loss), but it has no guard against single-node clusters. The alert has NOT been remediated since the prior investigation 2h52m ago — the metric has been flat at 0 for the entire 1620-minute lookback window, and the cluster reports healthy throughout.'
resource: observability/OpenBaoRaftQuorumAtRisk
tags:
    - runlore
    - incident
    - prometheusrule
    - observability
timestamp: "2026-09-12T18:50:02Z"
fingerprint: 3e2027e434606613eb106ea077d9475fd59b4e956918d059a8246eda903e20c9
confidence: 0.88
---

## Decision

- **why keep:** PRE-EXISTING FALSE POSITIVE (unchanged from occurrence #2): The alert rule `vault_autopilot_failure_tolerance{job="openbao"} < 1` fires on a single-node dev OpenBao cluster where failure_tolerance=0 is the mathematically correct and expected state. For a 1-node raft cluster, failure tolerance = (N-1)/2 = (1-1)/2 = 0. The rule is designed for multi-node HA clusters (3+ nodes, where failure_tolerance < 1 would indicate a real quorum loss), but it has no guard against single-node clusters. The alert has NOT been remediated since the prior investigation 2h52m ago — the metric has been flat at 0 for the entire 1620-minute lookback window, and the cluster reports healthy throughout.
- **confidence:** 88%

## Symptom

OpenBaoRaftQuorumAtRisk (pre-existing, occurrence #3) — false positive: alert threshold fires on single-node dev OpenBao cluster where failure_tolerance=0 is expected

Affected resource: PrometheusRule observability/OpenBaoRaftQuorumAtRisk

## Investigate

- alert_rule: expr=`vault_autopilot_failure_tolerance{job="openbao"} < 1` for=15m, state=firing
- query_metrics: `vault_autopilot_failure_tolerance{job="openbao"}` = 0 (currently firing threshold)
- query_metrics: `count(count by (peer_id) (vault_raft_storage_stats_applied_index{job="openbao"}))` = 1 — exactly ONE peer in the cluster
- query_metrics: `vault_raft_storage_stats_applied_index{job="openbao"}` has a single series with peer_id=`openbao-dev-cr28.europe-west4-a.c.ogenki-435905.internal` (note 'dev' in hostname)
- query_metrics: `vault_autopilot_healthy{job="openbao"}` = 1 — cluster reports itself as healthy
- query_metrics: `up{job="openbao"}` = 1 — node is up and being scraped
- query_metrics_range over 1620m: `vault_autopilot_failure_tolerance` first=0 last=0 min=0 max=0 — flat at 0 the entire window, not a recent change
- query_metrics_range over 1620m: `vault_autopilot_healthy` first=1 last=1 min=1 max=1 — healthy the entire window
- query_metrics_range over 1620m: `up{job="openbao"}` first=1 last=1 min=1 max=1 — up the entire window
- cloud_resource_health: compute instance `openbao-dev-cr28` (europe-west4-a): status=RUNNING
- what_changed (flux-system, 1620m): no changes to alert rules or OpenBao config — only zitadel.yaml suspend:true→false
- what_changed (security, 1620m): no changes to OpenBao-related resources
- No pods, StatefulSets, Deployments, or Services with app.kubernetes.io/name=openbao in the security namespace — OpenBao runs as an external GCE VM, not a Kubernetes workload
- pod_status (security): 3 pods — openbao-snapshot-29819760-9cc2z, -hfd8p, -pxhr8 — all Failed with exit 1, age=14h
- pod_logs (security, job-name=openbao-snapshot-29819760): 'INFO: Starting OpenBao backup to object storage...' → 'INFO: Authenticating with OpenBao via auth/jwt/gcp-0 as role openbao-snapshot...' → 'INFO: Requesting a snapshot via https://openbao.security.svc.cluster.local:8200' → 'ERROR: (gcloud.storage.cp) You do not currently have an active account selected.'
- pod_logs: all 3 pods show the same error pattern, first failing at 2026-09-12T04:01:26Z–04:02:46Z
- kube_events (security, object=openbao-snapshot-29819760): no Warning events in the window

## Cause

1. **PRE-EXISTING FALSE POSITIVE (unchanged from occurrence #2): The alert rule `vault_autopilot_failure_tolerance{job="openbao"} < 1` fires on a single-node dev OpenBao cluster where failure_tolerance=0 is the mathematically correct and expected state. For a 1-node raft cluster, failure tolerance = (N-1)/2 = (1-1)/2 = 0. The rule is designed for multi-node HA clusters (3+ nodes, where failure_tolerance < 1 would indicate a real quorum loss), but it has no guard against single-node clusters. The alert has NOT been remediated since the prior investigation 2h52m ago — the metric has been flat at 0 for the entire 1620-minute lookback window, and the cluster reports healthy throughout.** (88%)
2. **SECONDARY FINDING (not the cause of this alert): OpenBao snapshot backup CronJob pods are failing — 3 pods in Failed state for ~14h with GCS authentication errors: 'ERROR: (gcloud.storage.cp) You do not currently have an active account selected.' The snapshot process successfully authenticates to OpenBao and requests a snapshot, but fails when uploading to GCS object storage because the gcloud CLI has no active account. This means OpenBao backups are not being created.** (50%)

## Resolution

- Fix the alert rule to exclude single-node clusters. Options: (1) Add a condition like `vault_autopilot_failure_tolerance{job="openbao"} < 1 and on() (count by (job) (vault_raft_storage_stats_applied_index{job="openbao"}) > 1)` so the rule only fires when there are multiple peers AND failure tolerance is below 1; or (2) create a separate override rule for the dev single-node instance that inhibits this alert; or (3) if this dev cluster should eventually be multi-node, add a TODO to remove the override when it is. Until fixed, this alert will continue firing indefinitely on every evaluation cycle. (reversible=true)
- Fix the GCS authentication for the openbao-snapshot CronJob. The snapshot pod's gcloud CLI has no active account — either the service account key/Workload Identity is missing or misconfigured, or the gcloud auth setup step in the job's entrypoint is failing silently. Check: (1) the pod's service account and Workload Identity binding, (2) the job spec's env vars or volume mounts for GCP credentials, (3) whether the entrypoint script needs an explicit `gcloud auth activate-service-account` call. The OpenBao snapshot itself succeeds (the API request completes), so only the GCS upload step needs fixing. (reversible=true)

## Unresolved

- The alert rule's GitOps source file path is not directly accessible to determine whether the fix should be applied in the vmalert rules ConfigMap, a PrometheusRule CRD, or a Helm values override — the rule file is shown as /etc/vmalert/rules-out/rules-src-0/rules.yaml, which is a rendered path inside the vmalert pod. The on-call should locate the source in the observability/victoria-metrics-k8s-stack HelmRelease values or the flux source repo.
- The openbao-snapshot CronJob definition and its GCS auth configuration were not inspected directly (no matching GitOps object found in the security namespace via what_changed) — the exact fix for the backup auth issue requires examining the CronJob spec and its service account configuration, which may live in a Crossplane-managed or external Terraform-managed resource outside the Flux loop.

