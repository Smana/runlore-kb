---
type: Incident
title: Kustomization/runlore-faulttest HealthCheckFailed
description: The Kustomization failed its health check because the underlying Deployment could not pull its container image.
resource: flux-system/runlore-faulttest
tags:
    - runlore
    - incident
fingerprint: 703a76df3631e9685609e49a58b9d27f22bef3829f09972f307f5fa8bfb357f0
---

## Decision

- **why keep:** The Kustomization failed its health check because the underlying Deployment could not pull its container image.
- **confidence:** 90%

## Symptom

Kustomization/runlore-faulttest HealthCheckFailed

## Investigate

- Kustomization flux-system/runlore-faulttest Ready=False (HealthCheckFailed) due to timeout waiting for Deployment/runlore-test/faulttest status: 'InProgress'
- Pod faulttest-755cc49cb9-p5bnq in namespace runlore-test is in Pending state with ImagePullBackOff: failed to pull and unpack image "ghcr.io/smana/runlore-faulttest-doesnotexist:v0.0.0": failed to resolve image: failed to authorize: failed to fetch anonymous token: unexpected status from GET request to https://ghcr.io/token?scope=repository%3Asmana%2Frunlore-faulttest-doesnotexist%3Apull&service=ghcr.io: 403 Forbidden

## Cause

1. **The Kustomization failed its health check because the underlying Deployment could not pull its container image.** (90%)

## Resolution

- Correct the image reference in the Kustomization to a valid and accessible image. (reversible=false)

## Unresolved

- No recent changes were found for the Kustomization, suggesting the incorrect image reference was either pre-existing or introduced through an indirect change not captured by the `what_changed` tool for this specific resource.

