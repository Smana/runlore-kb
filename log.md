---
okf_version: "0.1"
type: Log
title: Change log
description: Chronological record of catalog changes (one line per ingest/curation).
---

# Change log

## 2026-08-02

* **Creation**: Added [HelmRelease victoria-logs InstallFailed: Cilium IPAM exhaustion on node ip-10-0-3-147 blocks DaemonSet pod + Flux…](incidents/helmrelease-victoria-logs-installfailed-cilium-ipam-exhaustion-on-node-ip-10-0-3-147-blocks-daemonset-pod-flux-790c3116.md).

## 2026-07-07

* **Creation**: Added [RunloreHistoryValidation — synthetic validation alert, no real incident (namespace/workload do not exist)](incidents/runlorehistoryvalidation-synthetic-validation-alert-no-real-incident-namespace-workload-do-not-exist-d383a759.md).

## 2026-07-10

* **Cleanup**: Removed `harborregistrydown.md` — a duplicate of [Harbor Registry Down due to IAM Access Key Quota Limit](harbor-registry-down-due-to-iam-access-key-quota-limit.md) (same IAM-quota incident, older resource-less format).
* **Fix**: [Harbor HelmRelease stuck InstallFailed](harbor-helmrelease-terminal-failed.md) and [Application/airflow Degraded](application-airflow-degraded.md) now carry the required `resource:` and `## Symptom` / `## Cause` sections, so the indexer stops dropping them.
* **CI**: Added `.github/workflows/validate-kb.yml` — every PR is now checked by RunLore's own `lore validate-kb`.

## 2026-07-13

* **Creation**: Added [envoy-gateway HelmRelease terminal-failed: chart v1.8.2 RBAC gap blocks InferencePool watch → healthz fails →…](incidents/envoy-gateway-helmrelease-terminal-failed-chart-v1-8-2-rbac-gap-blocks-inferencepool-watch-healthz-fails-c98cf34b.md).

## 2026-07-16

* **Creation**: Added [EKS managed control plane — KubeAPIDown/KubeControllerManagerDown/KubeSchedulerDown are false positives](eks-control-plane-down-alerts-false-positive.md).
* **Creation**: Added [LLMPlatformSemanticRouterDown on an LLM-free cluster — the opt-in LLM platform is suspended](llm-semantic-router-down-platform-suspended.md).
* **Creation**: Added [Crossplane KMS Alias stuck Synced=False — kms:CreateAlias denied by an unsatisfiable aws:RequestTag condition](crossplane-kms-createalias-requesttag-accessdenied.md).

## 2026-07-21

* **Fix**: [Harbor Registry Down due to IAM Access Key Quota Limit](harbor-registry-down-due-to-iam-access-key-quota-limit.md) gains `alert_resource: tooling/harbor` (runlore#318) — the cause recurred on the rebuilt cluster but fired as a HelmRelease-scoped trigger, which the pod-scoped `resource:` could not instant-recall.

## 2026-08-02

* **Revision**: [Harbor Registry Down due to IAM Access Key Quota Limit](harbor-registry-down-due-to-iam-access-key-quota-limit.md) — incident recurred a third time on the rebuilt cluster and was fixed end to end, so the entry's two `Unresolved` questions are now answered in place: the IAM user is `xplane-harbor`, and orphan keys are identified by a `CreateDate` predating the current cluster (or, decisively, by the `AccessKey` MR carrying an empty `EXTERNAL-NAME`, meaning this cluster owns none of them). Adds the verified 5-step remediation — including the two steps that were missing and cost the most time: `harbor-registry` needs an explicit `rollout restart` because the Deployment has already tripped `ProgressDeadlineExceeded` and will not pick the Secret up on its own, and a plain `flux reconcile hr --force` sufficed (`--reset` / `helm uninstall` were **not** needed despite 125+ failed AccessKey attempts). Adds a *Preventing the recurrence* key-sweep, and records the structural cause: the `xplane-harbor` IAM user outlives the cluster because the constitution withholds IAM delete rights from Crossplane, so every rebuild burns one of the two key slots and the third rebuild's Harbor cannot start. Supersedes RunLore PR #103. `last_validated: 2026-08-02`; tags widened for recall.
