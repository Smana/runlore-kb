---
type: Incident
title: Kustomization DependencyNotReady due to missing GitRepositories
description: Flux Kustomizations `crossplane-configuration` and `eks-pod-identities` are unable to reconcile because their underlying GitRepository sources (`infra-artifact` and `security-artifact`) cannot be found in the `flux-system` namespace.
tags:
    - runlore
    - incident
---

## Summary

Confidence 80%.

## Root causes
1. **Flux Kustomizations `crossplane-configuration` and `eks-pod-identities` are unable to reconcile because their underlying GitRepository sources (`infra-artifact` and `security-artifact`) cannot be found in the `flux-system` namespace.** (90%)
   - evidence: what_changed for flux-system/crossplane-configuration returned "error: get gitrepository flux-system/infra-artifact: gitrepositories.source.toolkit.fluxcd.io \"infra-artifact\" not found"
   - evidence: what_changed for flux-system/eks-pod-identities returned "error: get gitrepository flux-system/security-artifact: gitrepositories.source.toolkit.fluxcd.io \"security-artifact\" not found"
   - suggested: Investigate why the `infra-artifact` and `security-artifact` GitRepositories are missing or misconfigured in the `flux-system` namespace and rectify the issue. This may involve reapplying their manifests or checking their status. (reversible=true)

## Unresolved
- The specific reason why the `infra-artifact` and `security-artifact` GitRepositories are missing or not found is undetermined. Further investigation into recent changes affecting these GitRepository objects would be beneficial.
- Log queries with level filters (`level=error` or `level=warn`) failed to parse, preventing a deeper analysis of controller logs for specific errors related to source fetching or kustomization reconciliation.

