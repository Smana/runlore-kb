---
type: Incident
title: Kustomization DependencyNotReady due to Missing GitRepository Artifacts in flux-system
description: The Kustomization `flux-system/crds` is in a `DependencyNotReady` state because its required GitRepository artifact (`crds-artifact`) cannot be found. This also prevents the `flux-system/karpenter` Kustomization, which depends on `crds`, from becoming ready. Other critical `GitRepository` objects like `infra-artifact` and `apps-artifact` are also reported as missing within the `flux-system` namespace, indicating a broader issue with Flux's ability to access its source definitions.
tags:
    - runlore
    - incident
---

## Summary

Confidence 90%.

## Root causes
1. **The Kustomization `flux-system/crds` is in a `DependencyNotReady` state because its required GitRepository artifact (`crds-artifact`) cannot be found. This also prevents the `flux-system/karpenter` Kustomization, which depends on `crds`, from becoming ready. Other critical `GitRepository` objects like `infra-artifact` and `apps-artifact` are also reported as missing within the `flux-system` namespace, indicating a broader issue with Flux's ability to access its source definitions.** (90%)
   - evidence: `what_changed` for `karpenter` reported 'gitrepositories.source.toolkit.fluxcd.io "infra-artifact" not found'
   - evidence: `what_changed` for `crds` reported 'gitrepositories.source.toolkit.fluxcd.io "crds-artifact" not found'
   - evidence: `what_changed` for `flux-system` reported 'gitrepositories.source.toolkit.fluxcd.io "apps-artifact" not found'
   - evidence: The knowledge base indicates that 'dependency ... not ready' often stems from upstream Kustomization failures, such as missing CRDs or source access issues.
   - suggested: Investigate the status and configuration of the GitRepository objects named `infra-artifact`, `crds-artifact`, and `apps-artifact` in the `flux-system` namespace. Ensure they are correctly defined and accessible by Flux's `source-controller`. (reversible=true)

## Unresolved
- Unable to determine the specific change that led to the GitRepository objects being unavailable, as `what_changed` could not retrieve revision history for the missing artifacts.
- Could not query logs for the `source-controller` due to a connectivity issue with the logging backend, which would have provided more detailed error messages regarding GitRepository fetching failures.

