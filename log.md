---                                               
okf_version: "0.1"                                                            
type: Log
title: Change log                                                                                                                                                                                                
description: Chronological record of catalog changes (one line per ingest/curation).
---

# Change log
## 2026-07-07

* **Creation**: Added [RunloreHistoryValidation — synthetic validation alert, no real incident (namespace/workload do not exist)](incidents/runlorehistoryvalidation-synthetic-validation-alert-no-real-incident-namespace-workload-do-not-exist-d383a759.md).

## 2026-07-10

* **Cleanup**: Removed `harborregistrydown.md` — a duplicate of [Harbor Registry Down due to IAM Access Key Quota Limit](harbor-registry-down-due-to-iam-access-key-quota-limit.md) (same IAM-quota incident, older resource-less format).
* **Fix**: [Harbor HelmRelease stuck InstallFailed](harbor-helmrelease-terminal-failed.md) and [Application/airflow Degraded](application-airflow-degraded.md) now carry the required `resource:` and `## Symptom` / `## Cause` sections, so the indexer stops dropping them.
* **CI**: Added `.github/workflows/validate-kb.yml` — every PR is now checked by RunLore's own `lore validate-kb`.

