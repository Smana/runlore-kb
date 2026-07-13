---
type: Incident
title: Harbor HelmRelease InstallFailed — IAM AccessKey quota blocks harbor-registry pod startup
description: 'AWS IAM access-key quota (AccessKeysPerUser: 2) exceeded for the Crossplane-managed IAM user, so AccessKey/xplane-harbor cannot be created and the Secret tooling/xplane-harbor-access-key is missing the ''username'' key. The harbor-registry pod therefore fails with CreateContainerConfigError and stays Pending, causing the Harbor Helm install --wait to time out (InstallFailed).'
resource: tooling/harbor
tags:
    - runlore
    - incident
    - helmrelease
    - tooling
timestamp: "2026-07-13T10:34:48Z"
fingerprint: d6c699096cf69d7310f3799636e57cde3879fdefb67741a92cffcd70856611fa
confidence: 0.85
provenance:
    - AWS IAM quota AccessKeysPerUser exceeded on the IAM user backing Crossplane provider-aws (no GitOps change involved)
---

## Decision

- **why keep:** AWS IAM access-key quota (AccessKeysPerUser: 2) exceeded for the Crossplane-managed IAM user, so AccessKey/xplane-harbor cannot be created and the Secret tooling/xplane-harbor-access-key is missing the 'username' key. The harbor-registry pod therefore fails with CreateContainerConfigError and stays Pending, causing the Harbor Helm install --wait to time out (InstallFailed).
- **confidence:** 85%
- **provenance:** AWS IAM quota AccessKeysPerUser exceeded on the IAM user backing Crossplane provider-aws (no GitOps change involved)

## Symptom

Harbor HelmRelease InstallFailed — IAM AccessKey quota blocks harbor-registry pod startup

Affected resource: HelmRelease tooling/harbor

## Investigate

- kube_events on xplane-harbor: 'CannotCreateExternalResource (x247): ... operation error IAM: CreateAccessKey, StatusCode: 409, LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2'
- pod_status: harbor-registry-8b57b4d54-7lstk is Pending (3h) with 'CreateContainerConfigError: couldn't find key username in Secret tooling/xplane-harbor-access-key'
- kube_events on harbor-registry pod: 'Failed (x905): Error: couldn't find key username in Secret tooling/xplane-harbor-access-key'
- gitops_resource_status: HelmRelease tooling/harbor Ready=False (InstallFailed), message matches alert — timeout waiting on harbor-registry status InProgress
- what_changed: no GitOps changes found for tooling/harbor — incident is NOT caused by an application/chart change
- KB runbook 'harbor-registry-down-due-to-iam-access-key-quota-limit.md' describes this exact scenario and resolution

## Cause

1. **AWS IAM access-key quota (AccessKeysPerUser: 2) exceeded for the Crossplane-managed IAM user, so AccessKey/xplane-harbor cannot be created and the Secret tooling/xplane-harbor-access-key is missing the 'username' key. The harbor-registry pod therefore fails with CreateContainerConfigError and stays Pending, causing the Harbor Helm install --wait to time out (InstallFailed).** (85%) — change: AWS IAM quota AccessKeysPerUser exceeded on the IAM user backing Crossplane provider-aws (no GitOps change involved)

## Resolution

- An AWS admin deletes an old/unused access key for the IAM user associated with the Crossplane provider (provider-aws), so the quota drops below 2 and AccessKey/xplane-harbor can be created. Then reconcile the AccessKey/xplane-harbor Crossplane resource so it populates the Secret. Once harbor-registry starts, run 'flux -n tooling reconcile helmrelease harbor --reset' to clear the exhausted install-retry counters and let Flux reinstall cleanly. (reversible=true)

## Unresolved

- The specific IAM user name that has reached its AccessKeysPerUser quota (2) — needs to be identified in the AWS console/IAM to determine which old key to delete.
- Which of the two existing access keys is safe to delete (vs. actively in use by another service).
- Whether Flux has entered the terminal 'exceeded maximum retries' state — helm-controller logs returned no matching lines for 'harbor' (data gap); if terminal, 'flux reconcile helmrelease harbor --reset' is required after the Secret is fixed regardless.

## Citations

[1] AWS IAM quota AccessKeysPerUser exceeded on the IAM user backing Crossplane provider-aws (no GitOps change involved)

