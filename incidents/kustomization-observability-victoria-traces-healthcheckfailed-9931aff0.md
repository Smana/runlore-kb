---
type: Incident
title: Kustomization/observability-victoria-traces HealthCheckFailed
description: A transient issue with the `ArtifactGenerator/flux-system/monorepo-split` resource caused a cascading failure, leading to the `Kustomization/flux-system/observability-victoria-traces` failing its health check. The underlying issue with the `ArtifactGenerator` appears to have resolved, as evidenced by the downstream `HelmRelease/observability/victoria-traces` now being in a `Ready=True` state. The Kustomization, however, remains in a failed state and requires a manual reconciliation to clear the error.
resource: flux-system/observability-victoria-traces
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-07-07T07:04:32Z"
fingerprint: 9931aff01c8633a427c8684a28a9da213f8f2c68125902daea1d4730f223bbfc
confidence: 0.8
---

## Decision

- **why keep:** A transient issue with the `ArtifactGenerator/flux-system/monorepo-split` resource caused a cascading failure, leading to the `Kustomization/flux-system/observability-victoria-traces` failing its health check. The underlying issue with the `ArtifactGenerator` appears to have resolved, as evidenced by the downstream `HelmRelease/observability/victoria-traces` now being in a `Ready=True` state. The Kustomization, however, remains in a failed state and requires a manual reconciliation to clear the error.
- **confidence:** 80%

## Symptom

Kustomization/observability-victoria-traces HealthCheckFailed

Affected resource: Kustomization flux-system/observability-victoria-traces

## Investigate

- The Kustomization 'observability-victoria-traces' is in a HealthCheckFailed state, reporting a timeout waiting for the HelmRelease 'victoria-traces' to be ready.
- The 'victoria-traces' HelmRelease is currently in a Ready=True state, indicating the issue is no longer present with the HelmRelease itself.
- The gitops_tree output shows the root of the dependency chain is an ArtifactGenerator named 'monorepo-split' with a status of 'Ready=unknown', which, according to the knowledge base, can cause cascading failures in dependent resources.
- A knowledge base article, 'Flux bootstrap — Kustomizations DependencyNotReady until ArtifactGenerator artifacts exist', describes this exact scenario as a transient bootstrap race condition that typically self-resolves.

## Cause

1. **A transient issue with the `ArtifactGenerator/flux-system/monorepo-split` resource caused a cascading failure, leading to the `Kustomization/flux-system/observability-victoria-traces` failing its health check. The underlying issue with the `ArtifactGenerator` appears to have resolved, as evidenced by the downstream `HelmRelease/observability/victoria-traces` now being in a `Ready=True` state. The Kustomization, however, remains in a failed state and requires a manual reconciliation to clear the error.** (80%)

## Resolution

- Reconcile the Kustomization to force it to re-evaluate the health of its dependencies. (reversible=false)

