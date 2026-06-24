---
type: Incident
title: 'KubePodImagePullBackOff: Image Pull Authorization Failure'
description: The pod is failing to pull its container image because the image "ghcr.io/smana/runlore-nonexistent:v0.0.0-doesnotexist" does not exist or access is forbidden, resulting in a 403 Forbidden error during image pull authorization.
resource: runlore-test/broken-image-app
tags:
    - runlore
    - incident
fingerprint: bbf1549de4e628ba0f5c006e126ab52ea3375e9d4a6c575707b4602a8311f55b
---

## Decision

- **why keep:** The pod is failing to pull its container image because the image "ghcr.io/smana/runlore-nonexistent:v0.0.0-doesnotexist" does not exist or access is forbidden, resulting in a 403 Forbidden error during image pull authorization.
- **confidence:** 90%

## Symptom

KubePodImagePullBackOff: Image Pull Authorization Failure

## Investigate

- Pod "broken-image-app-77b7f5955c-7j26k" in namespace "runlore-test" is in `ImagePullBackOff` state.
- The pod status message indicates "failed to pull and unpack image ... failed to resolve image: failed to authorize: failed to fetch anonymous token: unexpected status ... 403 Forbidden".
- Kubernetes events show multiple "Failed to pull image" and "ImagePullBackOff" events for the same image with a 403 Forbidden error.

## Cause

1. **The pod is failing to pull its container image because the image "ghcr.io/smana/runlore-nonexistent:v0.0.0-doesnotexist" does not exist or access is forbidden, resulting in a 403 Forbidden error during image pull authorization.** (90%)

## Resolution

- Correct the image name for the "broken-image-app" deployment to a valid and accessible image. (reversible=false)

