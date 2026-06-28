---
type: Incident
title: Workload xplane-image-gallery not deployed due to missing GitOps resources
description: The GitOps resources (Kustomization or HelmRelease) for the "xplane-image-gallery" workload are missing in the "apps" namespace, preventing its deployment and the creation of its pods.
resource: apps/xplane-image-gallery
tags:
    - runlore
    - incident
timestamp: "2026-06-28T08:12:31Z"
fingerprint: a9b545170fc6f9ac35032783e27ebe2a480e502888fda664b80a21652f75ca6a
---

## Decision

- **why keep:** The GitOps resources (Kustomization or HelmRelease) for the "xplane-image-gallery" workload are missing in the "apps" namespace, preventing its deployment and the creation of its pods.
- **confidence:** 90%

## Symptom

Workload xplane-image-gallery not deployed due to missing GitOps resources

## Investigate

- pod_status for apps/xplane-image-gallery reports "no pods in namespace \"apps\" matching app=xplane-image-gallery".
- gitops_resource_status for Kustomization/apps/xplane-image-gallery reports "NOT FOUND".
- gitops_resource_status for HelmRelease/apps/xplane-image-gallery reports "NOT FOUND".

## Cause

1. **The GitOps resources (Kustomization or HelmRelease) for the "xplane-image-gallery" workload are missing in the "apps" namespace, preventing its deployment and the creation of its pods.** (90%)

## Resolution

- Investigate source control for the missing GitOps definition for xplane-image-gallery and restore it to the cluster. (reversible=false)

## Unresolved

- The precise reason for the absence of the xplane-image-gallery GitOps resources (e.g., whether they were explicitly deleted, never existed, or were misconfigured) could not be determined from the available change logs or controller logs specific to this resource.

