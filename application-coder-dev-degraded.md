---
type: Incident
title: Application/coder-dev Degraded
description: ArgoCD is unable to authenticate to the Git repository, preventing it from syncing the application state correctly and leading to a Degraded status.
resource: argocd/coder-dev
tags:
    - runlore
    - incident
timestamp: "2026-06-26T10:25:12Z"
fingerprint: cd0397ae2af59d828009d7e521cc04c81896d1f8c9395e66cc4e07ad66f4adc4
---

## Decision

- **why keep:** ArgoCD is unable to authenticate to the Git repository, preventing it from syncing the application state correctly and leading to a Degraded status.
- **confidence:** 90%

## Symptom

Application/coder-dev Degraded

## Investigate

- what_changed output for argocd/coder-dev: "diff error: clone git@github.com:Aqemia/engineering.git: invalid auth method"
- gitops_resource_status for argocd/coder-dev: Application is Ready=False (Degraded) and sync: OutOfSync
- gitops_resource_status for argocd/coder-dev: repoURL: git@github.com:Aqemia/engineering.git

## Cause

1. **ArgoCD is unable to authenticate to the Git repository, preventing it from syncing the application state correctly and leading to a Degraded status.** (90%)

## Resolution

- Investigate and correct the authentication method ArgoCD uses to access git@github.com:Aqemia/engineering.git. This likely involves checking the SSH key or credentials configured for the Git repository within ArgoCD. (reversible=true)

## Unresolved

- The specific content of the Git diff could not be retrieved due to the authentication error.

