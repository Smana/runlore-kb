---
type: Incident
title: Application/<app> Degraded
description: Argo CD is unable to authenticate to the Git repository "git@github.com:<org>/gitops-repo.git", leading to an OutOfSync application and unready pods.
resource: argocd/<app>
tags:
    - runlore
    - incident
timestamp: "2026-06-26T08:54:40Z"
fingerprint: debb29b9bde400b3a732c8e13ce0cfd4b044af6cdedab690d6a774bbee2e75a7
---

## Decision

- **why keep:** Argo CD is unable to authenticate to the Git repository "git@github.com:<org>/gitops-repo.git", leading to an OutOfSync application and unready pods.
- **confidence:** 90%

## Symptom

Application/<app> Degraded

## Investigate

- what_changed reported an "invalid auth method" error when trying to clone the repository.
- gitops_resource_status shows the Application/<app> is "OutOfSync" despite reporting "successfully synced (all tasks run)".
- pod_status shows the <app>-<pod-id> pod is "Running ready=0/1", indicating the application is not becoming ready, likely due to incorrect or missing configuration.
- kb_search indicates ArgoCD Application degraded status can be due to chart/values changes, but in this case, the root cause appears to be the inability to fetch the changes at all.

## Cause

1. **Argo CD is unable to authenticate to the Git repository "git@github.com:<org>/gitops-repo.git", leading to an OutOfSync application and unready pods.** (90%)

## Resolution

- Investigate and correct the Git authentication method (e.g., SSH key or token) used by Argo CD for accessing the "<org>/gitops-repo.git" repository. (reversible=true)

## Unresolved

- No logs could be retrieved from argocd-repo-server or argocd-application-controller due to tool error.

