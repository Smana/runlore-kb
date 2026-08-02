---
type: Incident
title: Kustomization/flux-system/eval-victim-broken ArtifactFailed — invalid spec.path references non-existent directory…
description: The Kustomization's spec.path is set to 'this/path/does/not/exist', a directory that does not exist within the ExternalArtifact/infra-artifact source. The source artifact itself is healthy (Ready=True), but the Kustomization cannot build because the configured path doesn't exist in the artifact contents. This is a static configuration error, not a regression — no recent GitOps change touched this Kustomization (what_changed returned no changes).
resource: flux-system/eval-victim-broken
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-08-02T10:17:33Z"
fingerprint: 0604fa650496da6486540d97b2f733b1963804bd62229b9282417b877772b95c
confidence: 0.85
---

## Decision

- **why keep:** The Kustomization's spec.path is set to 'this/path/does/not/exist', a directory that does not exist within the ExternalArtifact/infra-artifact source. The source artifact itself is healthy (Ready=True), but the Kustomization cannot build because the configured path doesn't exist in the artifact contents. This is a static configuration error, not a regression — no recent GitOps change touched this Kustomization (what_changed returned no changes).
- **confidence:** 85%

## Symptom

Kustomization/flux-system/eval-victim-broken ArtifactFailed — invalid spec.path references non-existent directory in source artifact

Affected resource: Kustomization flux-system/eval-victim-broken

## Investigate

- gitops_resource_status: Kustomization flux-system/eval-victim-broken Ready=False (ArtifactFailed), message: 'kustomization path not found: stat /tmp/kustomization-653102481/this/path/does/not/exist: no such file or directory'
- gitops_resource_status: ExternalArtifact flux-system/infra-artifact Ready=True (Succeeded, 'Artifact is ready') — the source is healthy, the path is wrong
- gitops_tree: Kustomization eval-victim-broken (Ready=False) → ExternalArtifact infra-artifact (Ready=True) → ArtifactGenerator monorepo-split (Ready=unknown). The failing node is the Kustomization itself, not its source.
- controller_logs (kustomize-controller): 'Reconciliation failed... error=kustomization path not found: stat /tmp/kustomization-653102481/this/path/does/not/exist: no such file or directory' at 10:12:42 and again at 10:13:43
- what_changed: 'no changes found for the given selector' — no recent GitOps change introduced this; it is a static misconfiguration
- incident_timeline: no preceding [git] or [flux] sync event for eval-victim-broken before the first ArtifactFailed at 10:12:42; other Kustomizations (security, observability, infrastructure, apps) synced successfully in the same window

## Cause

1. **The Kustomization's spec.path is set to 'this/path/does/not/exist', a directory that does not exist within the ExternalArtifact/infra-artifact source. The source artifact itself is healthy (Ready=True), but the Kustomization cannot build because the configured path doesn't exist in the artifact contents. This is a static configuration error, not a regression — no recent GitOps change touched this Kustomization (what_changed returned no changes).** (90%)

## Resolution

- Correct the Kustomization's spec.path to a valid path that exists within the ExternalArtifact/infra-artifact contents. Alternatively, if this Kustomization is not needed (the name 'eval-victim-broken' and path 'this/path/does/not/exist' suggest it may be a deliberately broken test resource), suspend it with 'flux suspend kustomization eval-victim-broken -n flux-system' or remove it from Git to stop the reconciliation churn and clear the alert. (reversible=true)

## Unresolved

- Whether 'eval-victim-broken' is a deliberately broken test/evaluation resource (the name and path 'this/path/does/not/exist' strongly suggest it) or an accidental misconfiguration — intent cannot be confirmed from available tooling. If intentional, no action is needed beyond alert tuning.

