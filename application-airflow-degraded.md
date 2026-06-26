---
type: Incident
title: Application/airflow Degraded
description: Argo CD Application "airflow" is degraded due to an inability to clone its Git repository. The error "invalid auth method" indicates a problem with the Git credentials used by Argo CD to access git@github.com:Aqemia/engineering.git.
resource: argocd/airflow
tags:
    - runlore
    - incident
timestamp: "2026-06-26T09:24:55Z"
fingerprint: 848fc4e66fdc8a0c849092000e240e1b0144a1d06a6e3666ea1dadcedde52cfc
---

## Decision

- **why keep:** Argo CD Application "airflow" is degraded due to an inability to clone its Git repository. The error "invalid auth method" indicates a problem with the Git credentials used by Argo CD to access git@github.com:Aqemia/engineering.git.
- **confidence:** 90%

## Symptom

Application/airflow Degraded

## Investigate

- what_changed for argocd/airflow showed "diff error: clone git@github.com:Aqemia/engineering.git: invalid auth method"
- gitops_resource_status for Application argocd/airflow is Ready=False (Degraded SyncFailed) with message "one or more synchronization tasks completed unsuccessfully (retried 5 times)."
- The Git repository URL is git@github.com:Aqemia/engineering.git

## Cause

1. **Argo CD Application "airflow" is degraded due to an inability to clone its Git repository. The error "invalid auth method" indicates a problem with the Git credentials used by Argo CD to access git@github.com:Aqemia/engineering.git.** (90%)

## Resolution

- Investigate and correct the Git credentials configured in Argo CD for accessing the Aqemia/engineering.git repository. (reversible=false)

## Unresolved

- Unable to query argocd-application-controller logs due to network error.

