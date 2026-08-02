---
type: Incident
title: 'Harbor HelmRelease InstallFailed: Crossplane AccessKey blocked by IAM AccessKeysPerUser quota (2/2)'
description: 'The Crossplane resource AccessKey/xplane-harbor cannot create an IAM access key because the associated IAM user already has 2 access keys, hitting the AWS IAM quota limit (AccessKeysPerUser: 2). Because the AccessKey never reconciles, the Kubernetes Secret tooling/xplane-harbor-access-key is never populated with the ''username''/''password'' keys. The harbor-registry Deployment references this Secret for its S3 storage credentials, so its container fails with CreateContainerConfigError: ''couldn''t find key username in Secret tooling/xplane-harbor-access-key''. The Deployment never becomes Ready, causing Helm''s --wait to time out and the HelmRelease to enter InstallFailed.'
resource: tooling/harbor
tags:
    - runlore
    - incident
    - helmrelease
    - tooling
timestamp: "2026-08-02T09:54:34Z"
fingerprint: d6c699096cf69d7310f3799636e57cde3879fdefb67741a92cffcd70856611fa
confidence: 0.9
provenance:
    - 'AWS IAM quota limit on the Crossplane provider''s IAM user (AccessKeysPerUser: 2 reached)'
---

## Decision

- **why keep:** The Crossplane resource AccessKey/xplane-harbor cannot create an IAM access key because the associated IAM user already has 2 access keys, hitting the AWS IAM quota limit (AccessKeysPerUser: 2). Because the AccessKey never reconciles, the Kubernetes Secret tooling/xplane-harbor-access-key is never populated with the 'username'/'password' keys. The harbor-registry Deployment references this Secret for its S3 storage credentials, so its container fails with CreateContainerConfigError: 'couldn't find key username in Secret tooling/xplane-harbor-access-key'. The Deployment never becomes Ready, causing Helm's --wait to time out and the HelmRelease to enter InstallFailed.
- **confidence:** 90%
- **provenance:** AWS IAM quota limit on the Crossplane provider's IAM user (AccessKeysPerUser: 2 reached)

## Symptom

Harbor HelmRelease InstallFailed: Crossplane AccessKey blocked by IAM AccessKeysPerUser quota (2/2)

Affected resource: HelmRelease tooling/harbor

## Investigate

- kube_events (tooling, 180m): 'AccessKey/xplane-harbor CannotCreateExternalResource (x125): async create failed: failed to create the resource: [{0 creating IAM Access Key (xplane-harbor): operation error IAM: CreateAccessKey, StatusCode: 409, LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2}]'
- pod_status (tooling): harbor-registry-7c5869dd4c-74gmq Pending ready=1/2 — 'CreateContainerConfigError: couldn't find key username in Secret tooling/xplane-harbor-access-key'
- gitops_resource_status: HelmRelease tooling/harbor Ready=False (InstallFailed) — 'Helm install failed for release tooling/harbor with chart harbor@1.18.3: timeout waiting for: [Deployment/tooling/harbor-registry status: InProgress]'
- incident_timeline: IAM quota error at 09:51:28Z → pod CreateContainerConfigError at 09:51:35Z — no Git change to Harbor in the window
- KB runbook 'Harbor Registry Down due to IAM Access Key Quota Limit' describes this exact failure scenario with matching symptoms
- gitops_resource_status: HelmRelease still Ready=False (InstallFailed) after 2+ hours of repeated failures
- KB runbook 'helmrelease-terminal-failed-exhausted-retries' notes that once retries are exhausted, Flux refuses to retry even after the root cause is fixed, requiring 'flux reconcile helmrelease --reset'
- helm-controller logs filtered by 'harbor' returned xplane-harbor-valkey entries, not harbor HelmRelease entries — could not directly confirm 'exceeded maximum retries' for the harbor release

## Cause

1. **The Crossplane resource AccessKey/xplane-harbor cannot create an IAM access key because the associated IAM user already has 2 access keys, hitting the AWS IAM quota limit (AccessKeysPerUser: 2). Because the AccessKey never reconciles, the Kubernetes Secret tooling/xplane-harbor-access-key is never populated with the 'username'/'password' keys. The harbor-registry Deployment references this Secret for its S3 storage credentials, so its container fails with CreateContainerConfigError: 'couldn't find key username in Secret tooling/xplane-harbor-access-key'. The Deployment never becomes Ready, causing Helm's --wait to time out and the HelmRelease to enter InstallFailed.** (90%) — change: AWS IAM quota limit on the Crossplane provider's IAM user (AccessKeysPerUser: 2 reached)
2. **The HelmRelease tooling/harbor may have exhausted its install retry budget (Flux helm-controller stops retrying after N failures, leaving the release terminal-failed). Even after the IAM quota is resolved and the Secret is populated, Flux may not automatically retry the install without a counter reset. This could not be directly confirmed from helm-controller logs (the resource filter returned the unrelated xplane-harbor-valkey release), but the InstallFailed state has persisted for ~2 hours with 125+ repeated AccessKey failures.** (20%)

## Resolution

- An AWS administrator must identify the IAM user associated with the Crossplane provider-config, list its access keys (aws iam list-access-keys), and delete an old/unused key to free a slot (aws iam delete-access-key). Then reconcile the Crossplane AccessKey/xplane-harbor resource so it creates the new key and populates the Secret. Once the Secret is populated, the harbor-registry pod will start and the HelmRelease install can proceed. (reversible=true)
- After the IAM quota is fixed and the Secret is populated, if the HelmRelease does not auto-recover, run 'flux -n tooling reconcile helmrelease harbor --reset' to clear exhausted retry counters and trigger a fresh install. If helm history shows only failed revisions, 'helm -n tooling uninstall harbor' first (PVCs persist), then the --reset reconcile. (reversible=true)

## Unresolved

- The name of the specific IAM user that has reached its access key quota — this must be identified in the AWS console/CLI by an administrator with IAM access.
- Which of the two existing access keys on that IAM user is safe to delete (one may be actively in use by another service).

## Citations

[1] AWS IAM quota limit on the Crossplane provider's IAM user (AccessKeysPerUser: 2 reached)

