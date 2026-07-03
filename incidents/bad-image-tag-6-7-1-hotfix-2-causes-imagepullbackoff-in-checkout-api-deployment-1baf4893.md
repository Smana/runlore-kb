---
type: Incident
title: Bad image tag '6.7.1-hotfix.2' causes ImagePullBackOff in checkout-api deployment
description: The checkout-api deployment is failing to pull an image due to an invalid image tag.
resource: flux-system/runlore-demo
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-07-03T21:24:41Z"
fingerprint: 1baf4893bac3c2c42763d15c92ed8cc6a152160801535d6a5ddc310f56d294ed
confidence: 0.9
---

## Decision

- **why keep:** The checkout-api deployment is failing to pull an image due to an invalid image tag.
- **confidence:** 90%

## Symptom

Bad image tag '6.7.1-hotfix.2' causes ImagePullBackOff in checkout-api deployment

Affected resource: Kustomization flux-system/runlore-demo

## Investigate

- A pod in the checkout-api Deployment is in ImagePullBackOff state with the error: `failed to pull and unpack image "ghcr.io/stefanprodan/podinfo:6.7.1-hotfix.2": failed to resolve image: ghcr.io/stefanprodan/podinfo:6.7.1-hotfix.2: not found`. This indicates the image tag is invalid.

## Cause

1. **The checkout-api deployment is failing to pull an image due to an invalid image tag.** (90%)

## Resolution

- The image tag for the checkout-api deployment should be reverted to a known-good tag. The current tag '6.7.1-hotfix.2' appears to be invalid or was removed from the 'ghcr.io/stefanprodan/podinfo' repository. An SRE with access to the image repository or the deployment configuration should identify a valid tag and update the deployment. (reversible=false)

