---
type: Incident
title: ImagePullBackOff due to incorrect image or credentials
description: The workload is failing to pull its container image due to a 403 Forbidden error. The image "ghcr.io/smana/runlore-nonexistent:v0.0.0-doesnotexist" likely does not exist or the image pull credentials are incorrect.
resource: runlore-test/broken-image-app
tags:
    - runlore
    - incident
fingerprint: 8ae88568435dde2d38d693d82cae1b555ab0c1f45fe5392dbcf6bf58170dd57a
---

## Decision

- **why keep:** The workload is failing to pull its container image due to a 403 Forbidden error. The image "ghcr.io/smana/runlore-nonexistent:v0.0.0-doesnotexist" likely does not exist or the image pull credentials are incorrect.
- **confidence:** 90%

## Symptom

ImagePullBackOff due to incorrect image or credentials

## Investigate

- pod_status shows "ImagePullBackOff: Back-off pulling image \"ghcr.io/smana/runlore-nonexistent:v0.0.0-doesnotexist\": ErrImagePull: failed to pull and unpack image \"ghcr.io/smana/runlore-nonexistent:v0.0.0-doesnotexist\": failed to resolve image: failed to authorize: failed to fetch anonymous token: unexpected status from GET request to https://ghcr.io/token?scope=repository%3Asmana%2Frunlore-nonexistent%3Apull&service=ghcr.io: 403 Forbidden"

## Cause

1. **The workload is failing to pull its container image due to a 403 Forbidden error. The image "ghcr.io/smana/runlore-nonexistent:v0.0.0-doesnotexist" likely does not exist or the image pull credentials are incorrect.** (90%)

## Resolution

- Correct the image name and/or tag in the broken-image-app workload definition. (reversible=false)

