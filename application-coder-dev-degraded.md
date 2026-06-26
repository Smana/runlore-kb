---
type: Incident
title: Application/coder-dev Degraded
description: ArgoCD Application coder-dev is degraded because ArgoCD is unable to clone the Git repository due to an invalid authentication method.
resource: argocd/coder-dev
tags:
    - runlore
    - incident
timestamp: "2026-06-26T11:36:45Z"
fingerprint: f25f80ebd65d9a91fa5faf4a46478e3df344503b79dd8d5c8b7ad5abd0065d2a
---

## Decision

- **why keep:** ArgoCD Application coder-dev is degraded because ArgoCD is unable to clone the Git repository due to an invalid authentication method.
- **confidence:** 90%

## Symptom

Application/coder-dev Degraded

## Investigate

- `what_changed` reported "clone git@github.com:<org>/gitops-repo.git: invalid auth method".
- `gitops_resource_status` shows Application argocd/coder-dev is `Ready=False (Degraded)` and `sync: OutOfSync`, indicating a failure to reconcile with the source repository.
- Frequent `OperationStarted` and `OperationCompleted` events for `coder-dev` suggest continuous attempts to sync, but the `OutOfSync` status indicates these attempts are not leading to a successful reconciliation with the intended state from Git.

## Cause

1. **ArgoCD Application coder-dev is degraded because ArgoCD is unable to clone the Git repository due to an invalid authentication method.** (90%)

## Resolution

- Investigate and correct the Git authentication method (e.g., SSH key or token) used by ArgoCD for the `git@github.com:<org>/gitops-repo.git` repository, specifically for the `coder-dev` application. (reversible=true)

## Unresolved

- The precise nature of the "invalid auth method" (e.g., expired key, incorrect permissions) could not be determined without further access to ArgoCD's Git credential configuration.
- Detailed logs from `argocd-repo-server` and `argocd-application-controller` could not be retrieved due to a tooling error ("no such host") and would have provided more granular information about the authentication failure and its immediate impact.

