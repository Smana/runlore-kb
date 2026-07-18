---
type: Incident
title: 'Harbor HelmRelease UpgradeFailed: IAM Access Key quota blocking harbor-registry credentials (not a capacity issue)'
description: 'IAM Access Key quota exceeded (AccessKeysPerUser: 2) for the Crossplane provider''s IAM user, preventing AccessKey/xplane-harbor from creating the credentials Secret that harbor-registry needs'
resource: tooling/harbor
tags:
    - runlore
    - incident
    - helmrelease
    - tooling
timestamp: "2026-07-18T14:19:13Z"
fingerprint: c5230eb8839081167d7423c7fa1a79730457b1b6d7681d5a4b87a4996cbf5180
confidence: 0.9
provenance:
    - No Git/config change — this is an AWS IAM quota limit hit by the Crossplane AccessKey/xplane-harbor resource
---

## Decision

- **why keep:** IAM Access Key quota exceeded (AccessKeysPerUser: 2) for the Crossplane provider's IAM user, preventing AccessKey/xplane-harbor from creating the credentials Secret that harbor-registry needs
- **confidence:** 90%
- **provenance:** No Git/config change — this is an AWS IAM quota limit hit by the Crossplane AccessKey/xplane-harbor resource

## Symptom

Harbor HelmRelease UpgradeFailed: IAM Access Key quota blocking harbor-registry credentials (not a capacity issue)

Affected resource: HelmRelease tooling/harbor

## Investigate

- pod_status: harbor-registry-56586ddb6-plhsx is Pending with CreateContainerConfigError: couldn't find key username in Secret tooling/xplane-harbor-access-key (ready=1/2, scheduled on node ip-10-0-10-135 with IP 100.64.46.200 — so NOT a scheduling/capacity issue)
- kube_events (Warning, x131 repeats, last 2026-07-18T14:15:19Z): AccessKey/xplane-harbor CannotCreateExternalResource: async create failed: creating IAM Access Key (xplane-harbor): StatusCode 409, LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2
- kube_events (Warning, x581 repeats, last 2026-07-18T14:15:13Z): Pod/harbor-registry-56586ddb6-plhsx Failed: couldn't find key username in Secret tooling/xplane-harbor-access-key
- kb_search matched 'Harbor Registry Down due to IAM Access Key Quota Limit' runbook at 95% confidence describing this exact failure chain
- what_changed found NO Git changes for tooling/harbor — ruling out a chart/config change as the trigger
- incident_timeline shows no harbor-related GitOps deploy; the only Git entry is crds-actions-runner-controller (unrelated) — same false correlation the past incident already rejected
- HelmRelease status: UpgradeFailed — timeout waiting for Deployment/harbor-registry InProgress. The harbor-registry Deployment cannot reach Ready because its pod is stuck in CreateContainerConfigError, so the Helm --wait times out

## Cause

1. **IAM Access Key quota exceeded (AccessKeysPerUser: 2) for the Crossplane provider's IAM user, preventing AccessKey/xplane-harbor from creating the credentials Secret that harbor-registry needs** (90%) — change: No Git/config change — this is an AWS IAM quota limit hit by the Crossplane AccessKey/xplane-harbor resource

## Resolution

- An AWS admin must delete an old/unused IAM access key for the Crossplane provider's IAM user to free a slot below the AccessKeysPerUser quota of 2. Then reconcile the AccessKey/xplane-harbor Crossplane resource so it creates the key and populates Secret tooling/xplane-harbor-access-key with the username key. Once the Secret is complete, harbor-registry's pod will start and the HelmRelease upgrade will succeed on Flux's next reconcile. (reversible=false)

## Unresolved

- Which of the two existing IAM access keys for the Crossplane provider's IAM user is safe to delete — an AWS admin must determine this in the IAM console before freeing the quota slot.

## Citations

[1] No Git/config change — this is an AWS IAM quota limit hit by the Crossplane AccessKey/xplane-harbor resource

