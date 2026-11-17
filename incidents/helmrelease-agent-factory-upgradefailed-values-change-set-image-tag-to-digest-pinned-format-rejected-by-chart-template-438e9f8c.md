---
type: Incident
title: 'HelmRelease/agent-factory UpgradeFailed: values change set image.tag to digest-pinned format rejected by chart template'
description: 'A Git change synced via Flux Kustomization flux-system/agent-factory at revision 79d80a7270af (11:47:49Z on 2026-11-17) modified the Helm values in ConfigMap agent-system/agent-factory-helm-values (or agent-factory-helm-values-sponsored) to set image.tag to a digest-pinned value in the format <version>@sha256:<digest>. The maku-factory chart''s deployment.yaml template explicitly rejects this format via a fail function at line 36:20, producing ''execution error at (maku-factory/templates/deployment.yaml:36:20): image.tag is <version>@sha256:<digest>'' on every upgrade attempt. The chart version (0.53.1+d3dd5292b5c4) and OCIRepository digest are unchanged from the last successful upgrade (v3 at 11:32:46Z), confirming the chart is fine and the values change is the sole cause.'
resource: agent-system/agent-factory
tags:
    - runlore
    - incident
    - helmrelease
    - agent-system
timestamp: "2026-11-17T11:54:25Z"
fingerprint: 438e9f8c0e608e4aa251d48dff744761863d039f5d91d404e797253cac23819a
confidence: 0.82
provenance:
    - 'flux Kustomization/agent-factory (sync): ..79d80a7270afbe3d407ebcc614f0d95f199d5a9a865c4e5de0fdfa86b0d116ff at 2026-11-17T11:47:49Z'
origin: runlore
---

## Decision

- **why keep:** A Git change synced via Flux Kustomization flux-system/agent-factory at revision 79d80a7270af (11:47:49Z on 2026-11-17) modified the Helm values in ConfigMap agent-system/agent-factory-helm-values (or agent-factory-helm-values-sponsored) to set image.tag to a digest-pinned value in the format <version>@sha256:<digest>. The maku-factory chart's deployment.yaml template explicitly rejects this format via a fail function at line 36:20, producing 'execution error at (maku-factory/templates/deployment.yaml:36:20): image.tag is <version>@sha256:<digest>' on every upgrade attempt. The chart version (0.53.1+d3dd5292b5c4) and OCIRepository digest are unchanged from the last successful upgrade (v3 at 11:32:46Z), confirming the chart is fine and the values change is the sole cause.
- **confidence:** 82%
- **provenance:** flux Kustomization/agent-factory (sync): ..79d80a7270afbe3d407ebcc614f0d95f199d5a9a865c4e5de0fdfa86b0d116ff at 2026-11-17T11:47:49Z

## Symptom

HelmRelease/agent-factory UpgradeFailed: values change set image.tag to digest-pinned format rejected by chart template

Affected resource: HelmRelease agent-system/agent-factory

## Investigate

- gitops_resource_status HelmRelease agent-system/agent-factory: Ready=False (UpgradeFailed), message 'execution error at (maku-factory/templates/deployment.yaml:36:20): image.tag is <version>@sha256:<digest>'
- resource_spec HelmRelease agent-factory: history shows v3 (status=deployed, lastDeployed 11:32:45Z, configDigest sha256:d767c01a...) succeeded with chartVersion 0.53.1+d3dd5292b5c4 and ociDigest sha256:d3dd5292...; the failed attempt has lastAttemptedConfigDigest sha256:ab95cd03... with the SAME lastAttemptedRevisionDigest sha256:d3dd5292... — chart unchanged, values changed
- what_changed(namespace=agent-system, name=agent-factory): 'flux Kustomization/agent-factory (sync): ..79d80a7270afbe3d407ebcc614f0d95f199d5a9a865c4e5de0fdfa86b0d116ff at 2026-11-17T11:47:49Z' — a Kustomization revision change occurred ~2 min before the first UpgradeFailed
- gitops_resource_status Kustomization flux-system/agent-factory: Ready=True (ReconciliationSucceeded), 'Applied revision: sha256:79d80a7270af...' — the Kustomization applied new ConfigMap values successfully at 11:47:49Z; event 'ConfigMap/agent-system/agent-factory-helm-values configured' at 11:32:45Z shows this ConfigMap is managed by this Kustomization
- controller_logs helm-controller agent-factory: at 11:49:51Z 'release out-of-sync with desired state: release config values changed' → 'running upgrade action' → 220ms later 'Reconciler error ... execution error at (maku-factory/templates/deployment.yaml:36:20): image.tag is <version>@sha256:<digest>' — the failure is triggered by values change, not chart change
- kube_events agent-system/agent-factory: UpgradeSucceeded at 11:32:46Z (v3) then UpgradeFailed starting at 11:49:51Z (8 failures) — all with identical error, confirming deterministic template rejection not transient
- gitops_resource_status OCIRepository agent-system/agent-factory: Ready=True, 'stored artifact for digest sha256:d3dd5292b5c4...' — chart source is healthy and unchanged
- resource_spec HelmRelease: spec.valuesFrom references ConfigMap/agent-factory-helm-values and ConfigMap/agent-factory-helm-values-sponsored — these are the values sources that the Kustomization manages

## Cause

1. **A Git change synced via Flux Kustomization flux-system/agent-factory at revision 79d80a7270af (11:47:49Z on 2026-11-17) modified the Helm values in ConfigMap agent-system/agent-factory-helm-values (or agent-factory-helm-values-sponsored) to set image.tag to a digest-pinned value in the format <version>@sha256:<digest>. The maku-factory chart's deployment.yaml template explicitly rejects this format via a fail function at line 36:20, producing 'execution error at (maku-factory/templates/deployment.yaml:36:20): image.tag is <version>@sha256:<digest>' on every upgrade attempt. The chart version (0.53.1+d3dd5292b5c4) and OCIRepository digest are unchanged from the last successful upgrade (v3 at 11:32:46Z), confirming the chart is fine and the values change is the sole cause.** (82%) — change: flux Kustomization/agent-factory (sync): ..79d80a7270afbe3d407ebcc614f0d95f199d5a9a865c4e5de0fdfa86b0d116ff at 2026-11-17T11:47:49Z

## Resolution

- Revert the Git change at revision 79d80a7270af in the source repository (OCIRepository flux-system/ai-native-ref) that modified the agent-factory Helm values ConfigMap to set image.tag to a digest-pinned format. Specifically, change image.tag back to a plain version string (e.g. v0.53.1) without the @sha256:<digest> suffix. If digest pinning is intentionally required, use the chart's designated mechanism for specifying a digest (e.g. image.digest field if supported) rather than embedding it in image.tag. After fixing the values, run 'flux -n flux-system reconcile kustomization agent-factory --force' to apply the corrected ConfigMap, then 'flux -n agent-system reconcile helmrelease agent-factory --force' to trigger a fresh upgrade. The HelmRelease currently has 8 failures with upgrade.remediation.retries=3; if it has entered a Stalled state after the fix, use 'flux -n agent-system reconcile helmrelease agent-factory --reset' to clear the failure counters. (reversible=true)

## Unresolved

- The exact contents of ConfigMaps agent-factory-helm-values and agent-factory-helm-values-sponsored could not be read (RBAC FORBIDDEN: 'configmaps is forbidden: User system:serviceaccount:runlore:runlore cannot get resource configmaps in API group in the namespace agent-system'). This prevented direct confirmation of the exact image.tag value that was set. The inference is based on the chart template error message and the configDigest change.
- The actual Git diff for Kustomization agent-factory revision 79d80a7270af was not available — what_changed showed the revision change line but no file-level diff. This prevented confirming which specific file/key in the values was changed.

## Citations

[1] flux Kustomization/agent-factory (sync): ..79d80a7270afbe3d407ebcc614f0d95f199d5a9a865c4e5de0fdfa86b0d116ff at 2026-11-17T11:47:49Z

