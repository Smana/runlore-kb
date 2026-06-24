---
type: Incident
title: KubePodImagePullBackOff
description: The pod is failing to pull an image due to a 403 Forbidden error, indicating the image `ghcr.io/smana/runlore-nonexistent:v0.0.0-doesnotexist` either does not exist or the repository credentials are incorrect. The image name itself suggests it is deliberately nonexistent.
resource: runlore-test/broken-image-app
tags:
    - runlore
    - incident
fingerprint: b38e55797816ff8afb8b5e374670facd25b334d4aa8a915495f74f37580329ea
---

## Decision

- **why keep:** The pod is failing to pull an image due to a 403 Forbidden error, indicating the image `ghcr.io/smana/runlore-nonexistent:v0.0.0-doesnotexist` either does not exist or the repository credentials are incorrect. The image name itself suggests it is deliberately nonexistent.
- **confidence:** 90%

## Symptom

KubePodImagePullBackOff

## Investigate

- pod_status output showing 'ErrImagePull: failed to pull and unpack image "ghcr.io/smana/runlore-nonexistent:v0.0.0-doesnotexist": failed to resolve image: failed to authorize: failed to fetch anonymous token: unexpected status from GET request to https://ghcr.io/token?scope=repository%3Asmana%2Frunlore-nonexistent%3Apull&service=ghcr.io: 403 Forbidden'

## Cause

1. **The pod is failing to pull an image due to a 403 Forbidden error, indicating the image `ghcr.io/smana/runlore-nonexistent:v0.0.0-doesnotexist` either does not exist or the repository credentials are incorrect. The image name itself suggests it is deliberately nonexistent.** (90%)

## Resolution

- Update the `broken-image-app` workload to reference a correct and accessible container image. (reversible=false)

## Unresolved

- The exact change that introduced the incorrect image reference could not be determined from `what_changed` for this workload.

