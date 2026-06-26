---
type: Incident
title: ArgoCD Application/airflow Degraded due to Git authentication failure
description: Git repository authentication failure preventing ArgoCD synchronization.
resource: argocd/airflow
tags:
    - runlore
    - incident
timestamp: "2026-06-26T10:24:46Z"
fingerprint: c40a7092205c07784cbbce6348755efa7e132e4487d56dba39174c0a105a5a7f
---

## Decision

- **why keep:** Git repository authentication failure preventing ArgoCD synchronization.
- **confidence:** 95%

## Symptom

ArgoCD Application/airflow Degraded due to Git authentication failure

## Investigate

- `what_changed` output showing 'diff error: clone git@github.com:Aqemia/engineering.git: invalid auth method'
- `gitops_resource_status` showing 'message: one or more synchronization tasks completed unsuccessfully (retried 5 times).'
- `gitops_resource_status` showing `repoURL: git@github.com:Aqemia/engineering.git`

## Cause

1. **Git repository authentication failure preventing ArgoCD synchronization.** (95%)

## Resolution

- Investigate and correct the authentication configuration for ArgoCD to access `git@github.com:Aqemia/engineering.git`. This might involve updating SSH keys, Git credentials, or repository access settings within ArgoCD. (reversible=true)

## Unresolved

- Log data from `argocd-repo-server` and `argocd-application-controller` could not be retrieved due to a 'no such host' error when querying the logs backend. This prevents further detailed analysis of the exact error messages during the git clone operation.

