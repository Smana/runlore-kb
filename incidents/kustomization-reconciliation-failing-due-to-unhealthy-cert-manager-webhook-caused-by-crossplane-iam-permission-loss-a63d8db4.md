---
type: Incident
title: Kustomization reconciliation failing due to unhealthy cert-manager webhook, caused by Crossplane IAM permission loss
description: Crossplane's AWS IAM permissions were revoked, causing a cascading failure of managed infrastructure in the 'security' namespace. This led to multiple critical admission webhooks (including cert-manager's) becoming unhealthy and unresponsive, which in turn blocked the 'zitadel' Kustomization from being applied.
resource: flux-system/zitadel
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-07-07T07:06:34Z"
fingerprint: a63d8db4c48e8f5421057f9c0f66a58a43f0a70dd390587ac2ef215f7f02dba7
confidence: 0.9
---

## Decision

- **why keep:** Crossplane's AWS IAM permissions were revoked, causing a cascading failure of managed infrastructure in the 'security' namespace. This led to multiple critical admission webhooks (including cert-manager's) becoming unhealthy and unresponsive, which in turn blocked the 'zitadel' Kustomization from being applied.
- **confidence:** 90%

## Symptom

Kustomization reconciliation failing due to unhealthy cert-manager webhook, caused by Crossplane IAM permission loss

Affected resource: Kustomization flux-system/zitadel

## Investigate

- Kubernetes events in the 'security' namespace show numerous 'AccessDeniedException' errors for Crossplane trying to perform AWS API calls, specifically 'kms:CreateAlias'. The user in the error is an assumed role from a Crossplane provider: 'arn:aws:sts::396740644681:assumed-role/mycluster-0-crossplane-20260707063144882300000013/eks-mycluster--provider-a-713ab247-a48b-46ce-8a1a-98da6e3bf3ca'.
- The 'cert-manager-webhook' pod in the 'security' namespace is not Ready (0/1) and is failing its liveness and readiness probes with the error 'context deadline exceeded', confirming it is unhealthy.
- Multiple other webhook-based applications in the 'security' namespace are also unhealthy, including 'external-secrets' and 'kyverno', indicating a systemic issue, not an isolated cert-manager problem.
- CloudTrail logs show recent 'RetireGrant' events on a KMS key, which could be related to the loss of permissions for Crossplane.

## Cause

1. **Crossplane's AWS IAM permissions were revoked, causing a cascading failure of managed infrastructure in the 'security' namespace. This led to multiple critical admission webhooks (including cert-manager's) becoming unhealthy and unresponsive, which in turn blocked the 'zitadel' Kustomization from being applied.** (90%)

## Resolution

- Investigate and restore the IAM permissions for the Crossplane provider's role in AWS. The role ARN can be found in the 'AccessDeniedException' events. Once permissions are restored, the Crossplane-managed resources should become healthy, followed by the webhooks, which will unblock the Kustomization reconciliation. (reversible=false)

