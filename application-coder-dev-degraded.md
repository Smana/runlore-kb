---
type: Incident
title: Application/coder-dev Degraded
description: ArgoCD Application "coder-dev" is degraded due to an invalid authentication method for accessing the Git repository.
resource: argocd/coder-dev
tags:
    - runlore
    - incident
timestamp: "2026-06-26T09:24:36Z"
fingerprint: 2d56d2fd5b9f5409ef3bbb5e4b208338f90df2cb4520691adbd0abf1225683b7
---

## Decision

- **why keep:** ArgoCD Application "coder-dev" is degraded due to an invalid authentication method for accessing the Git repository.
- **confidence:** 90%

## Symptom

Application/coder-dev Degraded

## Investigate

- `what_changed` output: "diff error: clone git@github.com:<org>/gitops-repo.git: invalid auth method"
- `gitops_resource_status` output: Application is Ready=False (Degraded) and sync: OutOfSync, with repoURL: git@github.com:<org>/gitops-repo.git

## Cause

1. **ArgoCD Application "coder-dev" is degraded due to an invalid authentication method for accessing the Git repository.** (90%)

## Resolution

- Verify and correct the authentication method ArgoCD uses to access the Git repository "git@github.com:<org>/gitops-repo.git". (reversible=false)

## Unresolved

- Logs from argocd-repo-server and argocd-application-controller are unavailable due to host lookup failure.

