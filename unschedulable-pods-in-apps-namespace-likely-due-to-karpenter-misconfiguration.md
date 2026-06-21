---
type: Incident
title: Unschedulable pods in `apps` namespace likely due to `karpenter` misconfiguration
description: The `apps` `Kustomization` may be misconfigured or missing, preventing any applications from being deployed or updated in the `apps` namespace.
tags:
    - runlore
    - incident
---

## Summary

Confidence 90%.

## Root causes
1. **The `apps` `Kustomization` may be misconfigured or missing, preventing any applications from being deployed or updated in the `apps` namespace.** (90%)
   - evidence: The `flux_tree` tool reports that the `apps` `Kustomization` is 'NOT FOUND'.
   - suggested: Investigate the GitRepository and Kustomization definitions for the `apps` namespace. (reversible=false)

## Unresolved
- The specific contents of the recent change to the `karpenter-nodepools` `Kustomization` could not be determined.
- No scheduling failure events could be found in the logs, despite multiple attempts with different query syntaxes.
- Rejected hypothesis: A recent change to the `karpenter-nodepools` configuration is likely preventing the `karpenter` autoscaler from provisioning new nodes, leading to unschedulable pods. — Rejecting. This is pure correlation. A change happened, but the diff was not read to establish a causal link to the failure. Empty logs are not strong evidence of a specific failure mode.

