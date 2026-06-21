---
type: Incident
title: Kustomization/crossplane-controller DependencyNotReady due to missing GitRepository
description: The Kustomization/crossplane-controller is failing due to a DependencyNotReady status for 'flux-system/crds'. Investigation revealed that the 'flux-system/crds' Kustomization cannot become ready because its associated GitRepository, 'flux-system/crds-artifact', is reported as not found.
tags:
    - runlore
    - incident
---

## Summary

Confidence 80%.

## Root causes
1. **The Kustomization/crossplane-controller is failing due to a DependencyNotReady status for 'flux-system/crds'. Investigation revealed that the 'flux-system/crds' Kustomization cannot become ready because its associated GitRepository, 'flux-system/crds-artifact', is reported as not found.** (90%)
   - evidence: Incident message: 'dependency 'flux-system/crds' is not ready.'
   - evidence: what_changed for 'crds' returned 'error: get gitrepository flux-system/crds-artifact: gitrepositories.source.toolkit.fluxcd.io "crds-artifact" not found'
   - evidence: what_changed for 'crossplane-controller' returned 'error: get gitrepository flux-system/infra-artifact: gitrepositories.source.toolkit.fluxcd.io "infra-artifact" not found'
   - suggested: Investigate why the 'gitrepository flux-system/crds-artifact' is missing or cannot be found. This could involve checking for accidental deletion, misconfiguration, or connectivity issues to the Git source. (reversible=false)

## Unresolved
- The underlying reason why the 'gitrepository flux-system/crds-artifact' is not found (e.g., accidental deletion, misconfiguration, or connectivity issue to Git provider).
- The logging system is unreachable, preventing log analysis and deeper investigation into controller errors.

