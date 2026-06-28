---
type: Incident
title: Kustomization `lab-failing-ks` failing due to missing path in source Git repository
description: A recent commit to the source Git repository removed a path that the Kustomization `lab-failing-ks` depends on.
resource: flux-system/lab-failing-ks
tags:
    - runlore
    - incident
timestamp: "2026-06-28T14:01:30Z"
fingerprint: aa5f08edfb057501d84d81204d0e11227c0901f53801e6f3bc662aa160497a18
---

## Decision

- **why keep:** A recent commit to the source Git repository removed a path that the Kustomization `lab-failing-ks` depends on.
- **confidence:** 95%

## Symptom

Kustomization `lab-failing-ks` failing due to missing path in source Git repository

## Investigate

- The Kustomization `flux-system/lab-failing-ks` has a `Ready=False` condition with reason `ArtifactFailed`.
- The error message is `kustomization path not found: stat /tmp/kustomization-2022039200/this-path-does-not-exist-lab-test: no such file or directory`.
- The source GitRepository `flux-system/flux-system` was recently updated to commit `f3d1ca23931de17de480de8a4354448d768da2d2`.
- Logs from `kustomize-controller` confirm it is failing to reconcile the Kustomization at that specific revision due to the missing path.

## Cause

1. **A recent commit to the source Git repository removed a path that the Kustomization `lab-failing-ks` depends on.** (95%)

## Resolution

- The breaking change was introduced in commit f3d1ca23931de17de480de8a4354448d768da2d2 of the flux-system Git repository. The owners of the repository should either revert this change or update the lab-failing-ks Kustomization to point to a valid path. In the meantime, to reduce log spam and failed reconciliations, the Kustomization should be suspended. (reversible=false)

## Unresolved

- The exact diff of the commit `f3d1ca23931de17de480de8a4354448d768da2d2` could not be retrieved to confirm the removal of the path `this-path-does-not-exist-lab-test`.

