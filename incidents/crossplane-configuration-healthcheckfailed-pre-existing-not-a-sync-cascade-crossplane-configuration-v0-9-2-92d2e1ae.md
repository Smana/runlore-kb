---
type: Incident
title: crossplane-configuration HealthCheckFailed (pre-existing) — NOT a sync cascade; Crossplane Configuration v0.9.2…
description: 'Crossplane Configuration package version skew: crossplane-configuration-aws was bumped to v0.9.2, which declares a dependency on crossplane-configuration-core >= v0.9.1, but the installed core package (smana-crossplane-configuration-core) is still at v0.7.2-pr35.0069ea6 — the version constraint cannot be satisfied, so Crossplane marks the AWS Configuration as Failed, which fails the Flux Kustomization''s health check.'
resource: flux-system/crossplane-configuration-aws
alert_resource: flux-system/crossplane-configuration
tags:
    - runlore
    - incident
    - configuration
    - flux-system
timestamp: "2026-10-08T04:57:04Z"
fingerprint: 92d2e1aec400a5b220c810dc3a1e06b9ddee6b461954490483d8b070a6f4b850
confidence: 0.88
provenance:
    - flux Kustomization/crossplane-configuration synced to revision latest@sha256:d3317f6fa1aeb777dc4c67a5b69ffaa4dbb5fa00ea0d0c360533b26d86d5df82 at 2026-10-08T04:47:05Z (first HealthCheckFailed at 04:48:05Z)
---

## Decision

- **why keep:** Crossplane Configuration package version skew: crossplane-configuration-aws was bumped to v0.9.2, which declares a dependency on crossplane-configuration-core >= v0.9.1, but the installed core package (smana-crossplane-configuration-core) is still at v0.7.2-pr35.0069ea6 — the version constraint cannot be satisfied, so Crossplane marks the AWS Configuration as Failed, which fails the Flux Kustomization's health check.
- **confidence:** 88%
- **provenance:** flux Kustomization/crossplane-configuration synced to revision latest@sha256:d3317f6fa1aeb777dc4c67a5b69ffaa4dbb5fa00ea0d0c360533b26d86d5df82 at 2026-10-08T04:47:05Z (first HealthCheckFailed at 04:48:05Z)

## Symptom

crossplane-configuration HealthCheckFailed (pre-existing) — NOT a sync cascade; Crossplane Configuration v0.9.2 requires core >= v0.9.1 but core is still v0.7.2

Affected resource: Configuration flux-system/crossplane-configuration-aws

## Investigate

- resource_spec on Configuration/crossplane-configuration-aws: status.conditions[Healthy].message = 'cannot resolve package dependencies: incompatible dependencies: existing package ghcr.io/smana/crossplane-configuration-core@v0.7.2-pr35.0069ea6 is incompatible with constraint >=v0.9.1', reason=UnhealthyPackageRevision, status=False
- resource_spec on Configuration/smana-crossplane-configuration-core: spec.package = ghcr.io/smana/crossplane-configuration-core:v0.7.2-pr35.0069ea6, status Healthy=True since 2026-10-07T18:15:08Z (not bumped)
- gitops_resource_status on Kustomization/crossplane-configuration: Ready=False, HealthCheckFailed, message = 'health check failed: failed early due to stalled resources: [Configuration/crossplane-configuration-aws status: Failed]' — failing repeatedly at 04:48, 04:49, 04:50, 04:51, 04:52, 04:55Z
- controller_logs (kustomize-controller): server-side apply completes successfully (Configuration/crossplane-configuration-aws='configured' then 'unchanged'), but health check fails because the Configuration object is in Failed state — not an apply or dependency cascade issue
- gitops_tree: all upstream dependencies (crossplane-providers, crossplane-controller, crds, namespaces, ExternalArtifacts) are Ready=True — this is NOT a cascade failure

## Cause

1. **Crossplane Configuration package version skew: crossplane-configuration-aws was bumped to v0.9.2, which declares a dependency on crossplane-configuration-core >= v0.9.1, but the installed core package (smana-crossplane-configuration-core) is still at v0.7.2-pr35.0069ea6 — the version constraint cannot be satisfied, so Crossplane marks the AWS Configuration as Failed, which fails the Flux Kustomization's health check.** (88%) — change: flux Kustomization/crossplane-configuration synced to revision latest@sha256:d3317f6fa1aeb777dc4c67a5b69ffaa4dbb5fa00ea0d0c360533b26d86d5df82 at 2026-10-08T04:47:05Z (first HealthCheckFailed at 04:48:05Z)

## Resolution

- Bump smana-crossplane-configuration-core to >= v0.9.1 so it satisfies the dependency constraint declared by crossplane-configuration-aws v0.9.2. The core Configuration is not in the crossplane-configuration Kustomization inventory (only crossplane-configuration-aws is) — find what manages the core package (possibly out-of-band or a different repo path) and update its spec.package to a compatible version. Alternatively, revert crossplane-configuration-aws to a version compatible with core v0.7.2. After fixing, the Configuration will reconcile to Healthy=True and the Kustomization health check will pass. (reversible=true)

## Unresolved

- What is the source of truth for smana-crossplane-configuration-core's package version? It needs to be bumped to >= v0.9.1 to match the AWS configuration's constraint, but the managing GitOps object/repo path could not be identified from available tools. A human needs to locate where the core Configuration's spec.package is declared and update it.

## Citations

[1] flux Kustomization/crossplane-configuration synced to revision latest@sha256:d3317f6fa1aeb777dc4c67a5b69ffaa4dbb5fa00ea0d0c360533b26d86d5df82 at 2026-10-08T04:47:05Z (first HealthCheckFailed at 04:48:05Z)

