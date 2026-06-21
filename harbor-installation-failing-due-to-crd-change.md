---
type: Incident
title: Harbor Installation Failing Due to CRD Change
description: The Harbor installation is failing due to a recent change to the 'crds-actions-runner-controller'.
tags:
    - runlore
    - incident
---

## Summary

Confidence 80%.

## Root causes
1. **The Harbor installation is failing due to a recent change to the 'crds-actions-runner-controller'.** (80%)
   - evidence: The Harbor HelmRelease is failing to install due to several of its components being in a crash loop.
   - evidence: The incident started after a recent change to the 'crds-actions-runner-controller'.
   - evidence: The crashing pods are not producing any logs, which suggests a low-level issue.
   - suggested: Revert the change to 'crds-actions-runner-controller'. (reversible=true)

## Unresolved
- The exact contents of the git diff for the 'crds-actions-runner-controller' change are unknown.

