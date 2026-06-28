---
type: Incident
title: ConfigTestDown
description: The `configtest` application's ConfigMap and likely the entire application were deleted from the GitOps repository, causing the pods to fail.
resource: sre-lab/configtest
tags:
    - runlore
    - incident
timestamp: "2026-06-28T14:00:13Z"
fingerprint: 53039235d783c4cd5ee8023456b0a7c268bdee305b21b6532071b5c1fb89bc42
---

## Decision

- **why keep:** The `configtest` application's ConfigMap and likely the entire application were deleted from the GitOps repository, causing the pods to fail.
- **confidence:** 95%

## Symptom

ConfigTestDown

## Investigate

- The `configtest` pod in the `sre-lab` namespace is in a `Pending` state with the reason `CreateContainerConfigError`.
- The error message for the pod is `configmap "configtest-settings" not found`.
- Kubernetes events for the pod confirm the missing `configtest-settings` ConfigMap.
- A change was detected for the `flux-system/apps` Kustomization, which manages applications in the cluster.
- No GitOps resources (`HelmRelease` or `Kustomization`) for `configtest` were found in the `sre-lab` or `flux-system` namespaces, suggesting the application was removed from GitOps management.

## Cause

1. **The `configtest` application's ConfigMap and likely the entire application were deleted from the GitOps repository, causing the pods to fail.** (95%)

## Resolution

- Re-add the configtest application and its associated `configtest-settings` ConfigMap to the Git repository under the management of the `flux-system/apps` Kustomization. (reversible=false)

## Unresolved

- The exact Git commit that removed the `configtest` application and its ConfigMap could not be determined.

