---
type: Incident
title: Application/airflow Degraded
description: ArgoCD Git repository authentication failure
resource: argocd/airflow
tags:
    - runlore
    - incident
timestamp: "2026-06-26T11:36:21Z"
fingerprint: 06cac07bc4255abe7aad24b780cebc108a9a9b44b224365ad65f5b14e0bf5ee5
---

## Decision

- **why keep:** ArgoCD Git repository authentication failure
- **confidence:** 90%
- **provenance:** ..e205e14d6e04f0376f44009c1937270f488ee5e1

## Symptom

Application/airflow Degraded

## Investigate

- `what_changed` reported "diff error: clone git@github.com:<org>/gitops-repo.git: invalid auth method".
- `gitops_resource_status` shows the Application in a `Degraded SyncFailed` state with the message "one or more synchronization tasks completed unsuccessfully (retried 5 times).".

## Cause

1. **ArgoCD Git repository authentication failure** (90%) — change: ..e205e14d6e04f0376f44009c1937270f488ee5e1

## Resolution

- Investigate and correct the Git authentication method (e.g., SSH key, Git credentials) configured in ArgoCD for the `<org>/gitops-repo.git` repository. After correcting, manually reconcile the `airflow` Application in ArgoCD. (reversible=true)

