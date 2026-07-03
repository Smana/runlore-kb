---
type: Incident
title: Kustomization runlore-demo failing health check due to missing secret for payment-api Deployment
description: A recent change to the runlore-demo application modified the ExternalSecret/payment-api-db to reference a source secret that does not exist. This prevents the application's required secret (payment-api-db) from being created, causing the payment-api pods to fail with CreateContainerConfigError.
resource: flux-system/runlore-demo
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-07-03T20:09:11Z"
fingerprint: 1baf4893bac3c2c42763d15c92ed8cc6a152160801535d6a5ddc310f56d294ed
confidence: 0.9
provenance:
    - 5b25d7141342d5dc5a179ed891808085a6ca773d
---

## Decision

- **why keep:** A recent change to the runlore-demo application modified the ExternalSecret/payment-api-db to reference a source secret that does not exist. This prevents the application's required secret (payment-api-db) from being created, causing the payment-api pods to fail with CreateContainerConfigError.
- **confidence:** 90%
- **provenance:** 5b25d7141342d5dc5a179ed891808085a6ca773d

## Symptom

Kustomization runlore-demo failing health check due to missing secret for payment-api Deployment

Affected resource: Kustomization flux-system/runlore-demo

## Investigate

- what_changed shows a recent sync for Kustomization/runlore-demo: 3b501eb..5b25d71
- pod_status in the runlore-demo namespace shows a payment-api pod in 'CreateContainerConfigError: secret "payment-api-db" not found'.
- Kubernetes events for ExternalSecret/runlore-demo/payment-api-db show 'UpdateFailed ... err: Secret does not exist'.
- The gitops_resource_status for Kustomization/flux-system/runlore-demo shows 'HealthCheckFailed' due to the 'payment-api' Deployment being stalled.

## Cause

1. **A recent change to the runlore-demo application modified the ExternalSecret/payment-api-db to reference a source secret that does not exist. This prevents the application's required secret (payment-api-db) from being created, causing the payment-api pods to fail with CreateContainerConfigError.** (90%) — change: 5b25d7141342d5dc5a179ed891808085a6ca773d

## Resolution

- Revert the Git commit 5b25d7141342d5dc5a179ed891808085a6ca773d that was applied to the runlore-demo Kustomization. (reversible=true)

## Citations

[1] 5b25d7141342d5dc5a179ed891808085a6ca773d

