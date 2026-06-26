---
type: Incident
title: Application/airflow Degraded
description: ArgoCD failed to clone the Git repository due to an invalid authentication method, preventing the 'airflow' Application from syncing.
resource: argocd/airflow
tags:
    - runlore
    - incident
timestamp: "2026-06-26T11:06:39Z"
fingerprint: ae867113f54da246e2f4c039844a00a4254eca5d97f3daa874f22aba54137c10
---

## Decision

- **why keep:** ArgoCD failed to clone the Git repository due to an invalid authentication method, preventing the 'airflow' Application from syncing.
- **confidence:** 95%

## Symptom

Application/airflow Degraded

## Investigate

- what_changed for argocd/airflow showed 'diff error: clone git@github.com:<org>/gitops-repo.git: invalid auth method'.
- gitops_resource_status for argocd/airflow showed Ready=False (Degraded SyncFailed) with message 'one or more synchronization tasks completed unsuccessfully (retried 5 times).'
- All ArgoCD core pods are running and healthy.

## Cause

1. **ArgoCD failed to clone the Git repository due to an invalid authentication method, preventing the 'airflow' Application from syncing.** (95%)

## Resolution

- Investigate and correct the Git authentication method configured in ArgoCD for the 'git@github.com:<org>/gitops-repo.git' repository. This might involve updating an SSH key or token. (reversible=false)

## Unresolved

- Could not query argocd-repo-server logs due to logging backend error.

