---
type: Incident
title: 'Kustomization reconciliation failure: Path not found'
description: The Kustomization 'flux-system/runlore-faulttest' is failing because its 'spec.path' references a non-existent directory 'tooling/base/runlore-faulttest' in the Git repository. The kustomize-controller cannot find this path, leading to reconciliation failures and health checks being canceled.
resource: flux-system/runlore-faulttest
tags:
    - runlore
    - incident
fingerprint: 0ac442000c2c2c22ad9a69728da80d84b2b4673eda2a42c02cd757717f07a05b
---

## Decision

- **why keep:** The Kustomization 'flux-system/runlore-faulttest' is failing because its 'spec.path' references a non-existent directory 'tooling/base/runlore-faulttest' in the Git repository. The kustomize-controller cannot find this path, leading to reconciliation failures and health checks being canceled.
- **confidence:** 90%
- **provenance:** ab881ce5758b93c667609548922beee2895d1d63

## Symptom

Kustomization reconciliation failure: Path not found

## Investigate

- flux_resource_status shows Warning ArtifactFailed: kustomization path not found: stat /tmp/kustomization-3281581015/tooling/base/runlore-faulttest: no such file or directory.
- what_changed indicates a recent sync of Kustomization/runlore-faulttest to commit ab881ce5758b93c667609548922beee2895d1d63, which likely introduced this issue.
- The kustomize-controller logs show the Kustomization itself being reconciled.

## Cause

1. **The Kustomization 'flux-system/runlore-faulttest' is failing because its 'spec.path' references a non-existent directory 'tooling/base/runlore-faulttest' in the Git repository. The kustomize-controller cannot find this path, leading to reconciliation failures and health checks being canceled.** (90%) — change: ab881ce5758b93c667609548922beee2895d1d63

## Resolution

- Revert the commit 'ab881ce5758b93c667609548922beee2895d1d63' or update the Kustomization 'flux-system/runlore-faulttest' to reference a valid path in the Git repository. (reversible=true)

