---
type: Incident
title: Harbor registry pod stuck in CreateContainerConfigError — Crossplane AccessKey/xplane-harbor blocked by IAM…
description: 'The Crossplane resource AccessKey/xplane-harbor cannot create an IAM access key because the target IAM user has already reached the AWS quota limit AccessKeysPerUser: 2. Crossplane''s CreateAccessKey call is rejected with LimitExceeded (repeated x582 times), so the Kubernetes Secret tooling/xplane-harbor-access-key is never populated with the required ''username'' key. The harbor-registry Deployment references this Secret, so the kubelet refuses to create the registry container with CreateContainerConfigError, leaving the pod Pending. Because the registry Deployment never becomes Ready, the Harbor HelmRelease install timed out and is now InstallFailed. This is a persistent condition (not a fresh change) — the container has been waiting >1h per the alert annotation.'
resource: tooling/harbor-registry
tags:
    - runlore
    - incident
    - pod
    - tooling
timestamp: "2026-07-10T19:10:37Z"
fingerprint: 9fc230d3534439c8251cc4119b4144dff3e8f586ebbb78308d809554ecf863e6
confidence: 0.88
---

## Decision

- **why keep:** The Crossplane resource AccessKey/xplane-harbor cannot create an IAM access key because the target IAM user has already reached the AWS quota limit AccessKeysPerUser: 2. Crossplane's CreateAccessKey call is rejected with LimitExceeded (repeated x582 times), so the Kubernetes Secret tooling/xplane-harbor-access-key is never populated with the required 'username' key. The harbor-registry Deployment references this Secret, so the kubelet refuses to create the registry container with CreateContainerConfigError, leaving the pod Pending. Because the registry Deployment never becomes Ready, the Harbor HelmRelease install timed out and is now InstallFailed. This is a persistent condition (not a fresh change) — the container has been waiting >1h per the alert annotation.
- **confidence:** 88%

## Symptom

Harbor registry pod stuck in CreateContainerConfigError — Crossplane AccessKey/xplane-harbor blocked by IAM AccessKeysPerUser quota

Affected resource: Pod tooling/harbor-registry-59598dbd57-ltkzw

## Investigate

- pod_status: harbor-registry-59598dbd57-ltkzw Pending, registry container: CreateContainerConfigError: couldn't find key username in Secret tooling/xplane-harbor-access-key
- kube_events (Warning, x2670): Pod/harbor-registry Failed: Error: couldn't find key username in Secret tooling/xplane-harbor-access-key
- kube_events (Warning, x582): AccessKey/xplane-harbor CannotCreateExternalResource: LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2 (StatusCode 409, IAM CreateAccessKey)
- gitops_resource_status HelmRelease tooling/harbor: Ready=False (InstallFailed) — timeout waiting for Deployment/tooling/harbor-registry status InProgress
- kb_search returned two matching runbooks ('Harbor Registry Down due to IAM Access Key Quota Limit' and 'HarborRegistryDown') both at 95% confidence describing this exact scenario
- what_changed for tooling namespace: only an unrelated crds-actions-runner-controller Kustomization change — no Harbor application change
- cloud_what_changed: only unrelated S3 PutBucketEncryption events for an openbao snapshot bucket — no IAM/ASG/EC2 change relevant to Harbor

## Cause

1. **The Crossplane resource AccessKey/xplane-harbor cannot create an IAM access key because the target IAM user has already reached the AWS quota limit AccessKeysPerUser: 2. Crossplane's CreateAccessKey call is rejected with LimitExceeded (repeated x582 times), so the Kubernetes Secret tooling/xplane-harbor-access-key is never populated with the required 'username' key. The harbor-registry Deployment references this Secret, so the kubelet refuses to create the registry container with CreateContainerConfigError, leaving the pod Pending. Because the registry Deployment never becomes Ready, the Harbor HelmRelease install timed out and is now InstallFailed. This is a persistent condition (not a fresh change) — the container has been waiting >1h per the alert annotation.** (88%)

## Resolution

- An AWS administrator must delete an old or unused access key on the IAM user associated with the Crossplane provider-aws, freeing a slot under the AccessKeysPerUser:2 quota. Once a slot is free, Crossplane will successfully reconcile AccessKey/xplane-harbor, populate the Secret with the username/password keys, and the harbor-registry pod will start. If Flux does not auto-retry the HelmRelease install, run 'flux -n tooling reconcile helmrelease harbor --reset' to clear the exhausted retry counters and trigger a fresh install. (reversible=false)

## Unresolved

- The specific IAM user name that has reached its AccessKeysPerUser:2 quota — must be identified in the AWS console/CLI by an administrator.
- Which of the two existing access keys on that IAM user is safe to delete — an admin must verify before deletion to avoid breaking another service that uses the other key.

