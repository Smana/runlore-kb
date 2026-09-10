---
type: Incident
title: 'Kustomization/apps ReconciliationFailed: App manifests add `.spec.externalSecrets[].store` field not declared in the…'
description: A coordinated Git change added a `store` field to `.spec.externalSecrets[]` in the App manifests (apps-artifact revision febc4acb...), but the App CRD schema (generated from the Crossplane XRD in the Configuration package) does not declare `store` as a valid field. Flux's server-side apply dry-run rejects the manifests with 'field not declared in schema'. The crossplane-configuration Kustomization applied Configuration/crossplane-configuration-aws as 'configured' (changed) at 20:59:19Z — 29 seconds before the apps Kustomization started failing at 20:59:48Z — but the CRD schema was not updated to include the `store` field. Both sources changed at nearly the same time, indicating a single commit that updated App manifests without a matching XRD schema update.
resource: flux-system/apps
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-09-10T21:07:51Z"
fingerprint: e5ca286bfd78ad40781685f6e83f51904b721c765fccd9e1ec21f3d522fc85f5
confidence: 0.78
provenance:
    - apps-artifact revision febc4acb8451cb61561eaa30753f5e8c87863f25171e4bb03c50dda5a791ddc1 (apps source) + crossplane-configuration/Configuration/crossplane-configuration-aws configured at 2026-09-10T20:59:19Z (infra-artifact source)
---

## Decision

- **why keep:** A coordinated Git change added a `store` field to `.spec.externalSecrets[]` in the App manifests (apps-artifact revision febc4acb...), but the App CRD schema (generated from the Crossplane XRD in the Configuration package) does not declare `store` as a valid field. Flux's server-side apply dry-run rejects the manifests with 'field not declared in schema'. The crossplane-configuration Kustomization applied Configuration/crossplane-configuration-aws as 'configured' (changed) at 20:59:19Z — 29 seconds before the apps Kustomization started failing at 20:59:48Z — but the CRD schema was not updated to include the `store` field. Both sources changed at nearly the same time, indicating a single commit that updated App manifests without a matching XRD schema update.
- **confidence:** 78%
- **provenance:** apps-artifact revision febc4acb8451cb61561eaa30753f5e8c87863f25171e4bb03c50dda5a791ddc1 (apps source) + crossplane-configuration/Configuration/crossplane-configuration-aws configured at 2026-09-10T20:59:19Z (infra-artifact source)

## Symptom

Kustomization/apps ReconciliationFailed: App manifests add `.spec.externalSecrets[].store` field not declared in the App CRD schema

Affected resource: Kustomization flux-system/apps

## Investigate

- gitops_resource_status Kustomization flux-system/apps: Ready=False (ReconciliationFailed) — 'App/apps/app-wizard dry-run failed: failed to create typed patch object (apps/app-wizard; cloud.ogenki.io/v1alpha1, Kind=App): errors: .spec.externalSecrets[0].store: field not declared in schema, .spec.externalSecrets[1].store: field not declared in schema'
- resource_spec App apps/app-wizard: live spec.externalSecrets[] entries have name, refreshInterval, remoteRef — NO store field; status Ready=True, Synced=True (Crossplane reconciled successfully at observedGeneration 5, lastTransitionTime 20:59:27Z) — the live resource is fine, the issue is the new Git source revision adds `store` which the CRD schema rejects
- kube_events flux-system/apps: ReconciliationFailed (x3) for app-wizard AND also 'App/apps/xplane-image-gallery dry-run failed: .spec.externalSecrets[0].store: field not declared in schema' — multiple App resources affected, confirming systemic manifest/schema mismatch
- controller_logs kustomize-controller crossplane-configuration: at 20:59:19Z 'Configuration/crossplane-configuration-aws: configured' (changed from previous 'unchanged') — the Crossplane Configuration package was updated 29s before the apps failure; at 21:00:20Z and later it shows 'unchanged' (update applied and stable)
- what_changed flux-system/apps: Kustomization/apps source revision advanced e4deaa26...→febc4acb... at 20:59:48Z — new apps-artifact revision with the `store` field in App manifests
- gitops_resource_status Kustomization flux-system/apps Events: '2026-09-10T20:58:48Z Normal ReconciliationSucceeded(x344)' — 344 prior successful reconciliations before the failure, confirming `store` was newly added in the latest revision
- gitops_tree Kustomization flux-system/apps: all dependencies Ready=True (tooling, apps-artifact, crds, crossplane-configuration) — failure is isolated to apps Kustomization dry-run, not a dependency cascade

## Cause

1. **A coordinated Git change added a `store` field to `.spec.externalSecrets[]` in the App manifests (apps-artifact revision febc4acb...), but the App CRD schema (generated from the Crossplane XRD in the Configuration package) does not declare `store` as a valid field. Flux's server-side apply dry-run rejects the manifests with 'field not declared in schema'. The crossplane-configuration Kustomization applied Configuration/crossplane-configuration-aws as 'configured' (changed) at 20:59:19Z — 29 seconds before the apps Kustomization started failing at 20:59:48Z — but the CRD schema was not updated to include the `store` field. Both sources changed at nearly the same time, indicating a single commit that updated App manifests without a matching XRD schema update.** (78%) — change: apps-artifact revision febc4acb8451cb61561eaa30753f5e8c87863f25171e4bb03c50dda5a791ddc1 (apps source) + crossplane-configuration/Configuration/crossplane-configuration-aws configured at 2026-09-10T20:59:19Z (infra-artifact source)

## Resolution

- Fix forward by updating the Crossplane XRD (in the crossplane-configuration source) to declare `store` as a field under `spec.externalSecrets[]` in the App composite resource schema, then push so the Configuration package regenerates the App CRD with the `store` field. Alternatively, if `store` was added to the App manifests prematurely/erroneously, revert the commit that added it. To stop reconciliation churn immediately: `flux suspend kustomization apps -n flux-system`. (reversible=true)

## Unresolved

- Whether the `store` field in the App manifests was intentionally added (and the XRD should have been updated to match) or was added erroneously — requires checking the Git commit history and the PR/intent. The fix direction (add `store` to XRD vs. remove `store` from manifests) depends on this.

## Citations

[1] apps-artifact revision febc4acb8451cb61561eaa30753f5e8c87863f25171e4bb03c50dda5a791ddc1 (apps source) + crossplane-configuration/Configuration/crossplane-configuration-aws configured at 2026-09-10T20:59:19Z (infra-artifact source)

