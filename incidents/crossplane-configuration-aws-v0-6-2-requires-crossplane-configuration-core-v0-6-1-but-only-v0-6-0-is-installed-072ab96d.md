---
type: Incident
title: crossplane-configuration-aws v0.6.2 requires crossplane-configuration-core >=v0.6.1, but only v0.6.0 is installed…
description: 'Crossplane Configuration package version mismatch: crossplane-configuration-aws was bumped to v0.6.2 (which declares a dependency on crossplane-configuration-core >=v0.6.1), but the core Configuration package smana-crossplane-configuration-core was NOT bumped and remains at v0.6.0. Crossplane cannot resolve the AWS package''s dependencies, marking its package revision Healthy=False (UnhealthyPackageRevision). The Flux Kustomization crossplane-configuration health-checks this Configuration object and therefore reports HealthCheckFailed.'
resource: crossplane-system/crossplane-configuration-aws
alert_resource: flux-system/crossplane-configuration
tags:
    - runlore
    - incident
    - configuration
    - crossplane-system
timestamp: "2026-09-10T21:34:30Z"
fingerprint: 072ab96d5ae2338a4bbe6f37dd089e3b8f8c58154fa8b598789d6caff5b2412c
confidence: 0.78
provenance:
    - infra-artifact revision 93a87be826a6 (synced 2026-09-10T21:28:34Z) — bumped crossplane-configuration-aws to v0.6.2 without a matching bump of crossplane-configuration-core
---

## Decision

- **why keep:** Crossplane Configuration package version mismatch: crossplane-configuration-aws was bumped to v0.6.2 (which declares a dependency on crossplane-configuration-core >=v0.6.1), but the core Configuration package smana-crossplane-configuration-core was NOT bumped and remains at v0.6.0. Crossplane cannot resolve the AWS package's dependencies, marking its package revision Healthy=False (UnhealthyPackageRevision). The Flux Kustomization crossplane-configuration health-checks this Configuration object and therefore reports HealthCheckFailed.
- **confidence:** 78%
- **provenance:** infra-artifact revision 93a87be826a6 (synced 2026-09-10T21:28:34Z) — bumped crossplane-configuration-aws to v0.6.2 without a matching bump of crossplane-configuration-core

## Symptom

crossplane-configuration-aws v0.6.2 requires crossplane-configuration-core >=v0.6.1, but only v0.6.0 is installed — package version mismatch causes Kustomization health check failure

Affected resource: Configuration crossplane-system/crossplane-configuration-aws

## Investigate

- resource_spec(Configuration/crossplane-configuration-aws): status.conditions[Healthy]=False, reason=UnhealthyPackageRevision, message='cannot resolve package dependencies: incompatible dependencies: existing package ghcr.io/smana/crossplane-configuration-core@v0.6.0 is incompatible with constraint >=v0.6.1', lastTransitionTime=2026-09-10T21:28:39Z, observedGeneration=3
- resource_spec(Configuration/smana-crossplane-configuration-core): spec.package=ghcr.io/smana/crossplane-configuration-core:v0.6.0, status.conditions[Healthy]=True at v0.6.0, observedGeneration=1, lastTransitionTime=2026-09-10T10:06:41Z (unchanged for ~11h while the AWS package was bumped)
- resource_spec(Configuration/crossplane-configuration-aws): spec.package=ghcr.io/smana/crossplane-configuration-aws:v0.6.2 — the AWS package was bumped to v0.6.2 in the latest revision (93a87be826a6)
- controller_logs(kustomize-controller, crossplane-configuration): at 21:28:35Z 'server-side apply completed' with Configuration/crossplane-configuration-aws='configured' (revision 93a87be826a6), then at 21:29:36Z error='health check failed after 31.483769ms: failed early due to stalled resources: [Configuration/crossplane-configuration-aws status: Failed]'
- incident_timeline: infra-artifact synced to 93a87be826a6 at 21:28:34Z → AWS Configuration health went False at 21:28:39Z → first HealthCheckFailed event at 21:29:36Z
- gitops_tree(Kustomization/crossplane-configuration): all Flux dependencies (crossplane-providers, crossplane-controller, crds, namespaces, infra-artifact) are Ready=True — the failure is NOT a Flux dependency cascade
- pod_status(crossplane-system): all 10 Crossplane pods Running/Ready with 0 current restarts — no runtime/pod-level issue; this is purely a Crossplane package dependency resolution failure

## Cause

1. **Crossplane Configuration package version mismatch: crossplane-configuration-aws was bumped to v0.6.2 (which declares a dependency on crossplane-configuration-core >=v0.6.1), but the core Configuration package smana-crossplane-configuration-core was NOT bumped and remains at v0.6.0. Crossplane cannot resolve the AWS package's dependencies, marking its package revision Healthy=False (UnhealthyPackageRevision). The Flux Kustomization crossplane-configuration health-checks this Configuration object and therefore reports HealthCheckFailed.** (78%) — change: infra-artifact revision 93a87be826a6 (synced 2026-09-10T21:28:34Z) — bumped crossplane-configuration-aws to v0.6.2 without a matching bump of crossplane-configuration-core

## Resolution

- Bump the smana-crossplane-configuration-core Configuration package from v0.6.0 to >= v0.6.1 (ideally v0.6.2 to match the AWS package family) in the Git manifest that defines it, then let Flux reconcile. The AWS Configuration v0.6.2 declares dependency constraint >=v0.6.1 on the core package, which v0.6.0 does not satisfy. After the core package is upgraded, Crossplane will resolve the AWS package's dependencies and the Kustomization health check will pass. (reversible=true)

## Unresolved

- Which Git path/Kustomization owns the smana-crossplane-configuration-core manifest — the crossplane-configuration Kustomization's SSA output does not include it (it applies the AWS Configuration but not the core one). A human should locate the manifest defining spec.package: ghcr.io/smana/crossplane-configuration-core:v0.6.0 and bump it there. It may be in the crossplane-providers or infrastructure Kustomization's source path.

## Citations

[1] infra-artifact revision 93a87be826a6 (synced 2026-09-10T21:28:34Z) — bumped crossplane-configuration-aws to v0.6.2 without a matching bump of crossplane-configuration-core

