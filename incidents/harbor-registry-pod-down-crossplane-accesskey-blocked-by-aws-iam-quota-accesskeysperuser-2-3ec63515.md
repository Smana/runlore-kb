---
type: Incident
title: 'harbor-registry pod down — Crossplane AccessKey blocked by AWS IAM quota (AccessKeysPerUser: 2)'
description: 'The Crossplane resource AccessKey/xplane-harbor cannot create the IAM access key Harbor needs because the associated IAM user has already hit the AWS quota of 2 access keys per user (LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2). With the key never created, the Secret tooling/xplane-harbor-access-key is missing its `username` key, so the harbor-registry container fails to start with CreateContainerConfigError and the pod stays Pending.'
resource: tooling/harbor-registry-59598dbd57-ltkzw
tags:
    - runlore
    - incident
    - pod
    - tooling
timestamp: "2026-07-10T13:12:23Z"
fingerprint: 3ec635159718d9781604e7a663b869bab7d6c52fb06eaed98613eecbd779a4df
confidence: 0.88
provenance:
    - AWS IAM quota limit on the Crossplane provider's IAM user — not a GitOps change
---

## Decision

- **why keep:** The Crossplane resource AccessKey/xplane-harbor cannot create the IAM access key Harbor needs because the associated IAM user has already hit the AWS quota of 2 access keys per user (LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2). With the key never created, the Secret tooling/xplane-harbor-access-key is missing its `username` key, so the harbor-registry container fails to start with CreateContainerConfigError and the pod stays Pending.
- **confidence:** 88%
- **provenance:** AWS IAM quota limit on the Crossplane provider's IAM user — not a GitOps change

## Symptom

harbor-registry pod down — Crossplane AccessKey blocked by AWS IAM quota (AccessKeysPerUser: 2)

Affected resource: Pod tooling/harbor-registry-59598dbd57-ltkzw

## Investigate

- pod_status: harbor-registry-59598dbd57-ltkzw is Pending (1/2 ready) with `CreateContainerConfigError: couldn't find key username in Secret tooling/xplane-harbor-access-key`
- kube_events: AccessKey/xplane-harbor emits `CannotCreateExternalResource` (x222 repeats) — `LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2` (IAM CreateAccessKey returned HTTP 409)
- kube_events: the harbor-registry Pod has been failing the same way since 2026-07-10T13:07Z, repeating 1005 times — consistent, non-transient
- kb_search runbook 'HarborRegistryDown' documents this exact symptom-to-cause chain at 95% confidence
- All other Harbor components (harbor-core, jobservice, nginx, portal, trivy, valkey) are Running/Ready — this is NOT a cluster-wide capacity issue; only harbor-registry depends on the missing Secret

## Cause

1. **The Crossplane resource AccessKey/xplane-harbor cannot create the IAM access key Harbor needs because the associated IAM user has already hit the AWS quota of 2 access keys per user (LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2). With the key never created, the Secret tooling/xplane-harbor-access-key is missing its `username` key, so the harbor-registry container fails to start with CreateContainerConfigError and the pod stays Pending.** (88%) — change: AWS IAM quota limit on the Crossplane provider's IAM user — not a GitOps change

## Resolution

- An administrator with AWS access must delete an old/unused access key on the IAM user associated with the Crossplane provider, then reconcile AccessKey/xplane-harbor so Crossplane can create the new key and populate the Secret. Once the Secret has the `username` key, the harbor-registry pod will start on its own. (reversible=false)

## Unresolved

- The name of the specific IAM user under the Crossplane provider config that has reached its access-key quota — must be identified in the AWS console / Crossplane ProviderConfig.
- Which of the two existing access keys on that IAM user is safe to delete (one may be actively in use by another workload).

## Citations

[1] AWS IAM quota limit on the Crossplane provider's IAM user — not a GitOps change

