---
type: Incident
title: Kustomization/security DependencyNotReady due to missing GitRepository
description: The GitRepository resource flux-system/security-artifact is missing or misconfigured, which is a dependency for the flux-system/security and flux-system/eks-pod-identities Kustomizations.
tags:
    - runlore
    - incident
---

## Summary

Confidence 90%.

## Root causes
1. **The GitRepository resource flux-system/security-artifact is missing or misconfigured, which is a dependency for the flux-system/security and flux-system/eks-pod-identities Kustomizations.** (90%)
   - evidence: what_changed(name='security', namespace='flux-system') output: "error: get gitrepository flux-system/security-artifact: gitrepositories.source.toolkit.fluxcd.io \"security-artifact\" not found"
   - evidence: what_changed(name='eks-pod-identities', namespace='flux-system') output: "error: get gitrepository flux-system/security-artifact: gitrepositories.source.toolkit.fluxcd.io \"security-artifact\" not found"
   - evidence: The alert message explicitly states 'dependency 'flux-system/eks-pod-identities' is not ready', and both Kustomizations depend on the 'security-artifact' GitRepository.
   - evidence: Knowledge base entry 'kustomization-reconciliation-failure.md' highlights 'dependency ... not ready' as a symptom of upstream issues.
   - suggested: Investigate and resolve the missing or misconfigured GitRepository resource flux-system/security-artifact. (reversible=true)

## Unresolved
- The logs query tool failed with 'dial tcp: lookup victoria-logs-victoria-logs-single-server.observability.svc on 172.20.0.10:53: no such host', preventing log analysis.
- Metrics queries for Kustomizations returned 'no series matched', preventing metric-based investigation.
- The 'flux-system/apps-artifact' GitRepository is also reported as not found during a broader 'what_changed' call for the namespace, indicating a potentially broader issue with GitRepository resources in the 'flux-system' namespace that might require further investigation.

