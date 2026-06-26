---
type: Incident
title: Application/coder-dev Degraded
description: Argo CD failed to clone the Git repository due to an invalid authentication method, preventing it from fetching the desired state of the application. Although sync operations are reported as "succeeded," the application remains Degraded and OutOfSync, suggesting Argo CD is not deploying the correct or latest configuration.
resource: argocd/coder-dev
tags:
    - runlore
    - incident
timestamp: "2026-06-26T10:55:01Z"
fingerprint: 0fb5114fc6816f7f1dea54a48459844c92f6d46bd201c04961d8a5abef8bcf61
---

## Decision

- **why keep:** Argo CD failed to clone the Git repository due to an invalid authentication method, preventing it from fetching the desired state of the application. Although sync operations are reported as "succeeded," the application remains Degraded and OutOfSync, suggesting Argo CD is not deploying the correct or latest configuration.
- **confidence:** 90%

## Symptom

Application/coder-dev Degraded

## Investigate

- `what_changed` reported "diff error: clone git@github.com:Aqemia/engineering.git: invalid auth method" for Application/coder-dev.
- `gitops_resource_status` shows Application/coder-dev as `Ready=False (Degraded)` and `sync: OutOfSync`, despite numerous "Partial sync operation ... succeeded" events.

## Cause

1. **Argo CD failed to clone the Git repository due to an invalid authentication method, preventing it from fetching the desired state of the application. Although sync operations are reported as "succeeded," the application remains Degraded and OutOfSync, suggesting Argo CD is not deploying the correct or latest configuration.** (90%)

## Resolution

- Investigate and correct the authentication method for Argo CD to access the Git repository: `git@github.com:Aqemia/engineering.git`. (reversible=false)

## Unresolved

- The exact content of the change that triggered the Degraded state could not be inspected due to the Git authentication error.

