---
type: Incident
title: Harbor registry down — Crossplane AccessKey blocked by AWS IAM AccessKeysPerUser quota (2)
description: 'The Crossplane AccessKey/xplane-harbor resource cannot create an IAM access key because the IAM user has already exhausted its AWS quota of AccessKeysPerUser: 2 (CreateAccessKey returns HTTP 409 LimitExceeded). As a result, the Secret tooling/xplane-harbor-access-key (which the harbor-registry container expects to mount, with keys username/password) is never created or is missing the required keys. The harbor-registry pod''s registry container therefore fails with CreateContainerConfigError: couldn''t find key username in Secret tooling/xplane-harbor-access-key and stays Pending (ready=1/2), so the Harbor registry endpoint is not serving.'
resource: tooling/harbor-registry
tags:
    - runlore
    - incident
    - pod
    - tooling
timestamp: "2026-07-10T21:29:12Z"
fingerprint: 69a522825fb996b780a148e31e42db3e13a1cddb8d7a89a826c18016c952661f
confidence: 0.75
provenance:
    - 'AWS IAM quota state (AccessKeysPerUser: 2) — not a GitOps change; the cluster had no Git changes to the tooling/harbor workload'
---

## Decision

- **why keep:** The Crossplane AccessKey/xplane-harbor resource cannot create an IAM access key because the IAM user has already exhausted its AWS quota of AccessKeysPerUser: 2 (CreateAccessKey returns HTTP 409 LimitExceeded). As a result, the Secret tooling/xplane-harbor-access-key (which the harbor-registry container expects to mount, with keys username/password) is never created or is missing the required keys. The harbor-registry pod's registry container therefore fails with CreateContainerConfigError: couldn't find key username in Secret tooling/xplane-harbor-access-key and stays Pending (ready=1/2), so the Harbor registry endpoint is not serving.
- **confidence:** 75%
- **provenance:** AWS IAM quota state (AccessKeysPerUser: 2) — not a GitOps change; the cluster had no Git changes to the tooling/harbor workload

## Symptom

Harbor registry down — Crossplane AccessKey blocked by AWS IAM AccessKeysPerUser quota (2)

Affected resource: Pod tooling/harbor-registry-59598dbd57-ltkzw

## Investigate

- kube_events: Warning AccessKey/xplane-harbor CannotCreateExternalResource (x722): async create failed: ... operation error IAM: CreateAccessKey, ... StatusCode: 409, ... LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2
- pod_status: harbor-registry-59598dbd57-ltkzw Pending ready=1/2 — registry: CreateContainerConfigError: couldn't find key username in Secret tooling/xplane-harbor-access-key
- kube_events: Pod/harbor-registry-59598dbd57-ltkzw Failed (x3296): Error: couldn't find key username in Secret tooling/xplane-harbor-access-key
- kb_search matched runbook 'HarborRegistryDown due to IAM Access Key Quota Limit' describing this exact failure and the IAM-quota fix
- HelmRelease tooling/harbor Ready=False (InstallFailed): timeout waiting for Deployment/harbor-registry — the install cannot complete because the registry pod is stuck Pending on the missing Secret

## Cause

1. **The Crossplane AccessKey/xplane-harbor resource cannot create an IAM access key because the IAM user has already exhausted its AWS quota of AccessKeysPerUser: 2 (CreateAccessKey returns HTTP 409 LimitExceeded). As a result, the Secret tooling/xplane-harbor-access-key (which the harbor-registry container expects to mount, with keys username/password) is never created or is missing the required keys. The harbor-registry pod's registry container therefore fails with CreateContainerConfigError: couldn't find key username in Secret tooling/xplane-harbor-access-key and stays Pending (ready=1/2), so the Harbor registry endpoint is not serving.** (75%) — change: AWS IAM quota state (AccessKeysPerUser: 2) — not a GitOps change; the cluster had no Git changes to the tooling/harbor workload

## Resolution

- An AWS administrator must identify the IAM user associated with the Crossplane provider-aws IAM credentials, list its access keys (aws iam list-access-keys --user-name <user>), and delete an old/unused one (aws iam delete-access-key --access-key-id <AKIA...> --user-name <user>) to free a quota slot. Once a slot is available, Crossplane will reconcile AccessKey/xplane-harbor and create/patch the Secret tooling/xplane-harbor-access-key with username/password; the harbor-registry pod will then start on its own. After the Secret exists, if Flux does not retry automatically, run: flux -n tooling reconcile helmrelease harbor --reset (clears exhausted install-retry counters). Verify with: kubectl -n tooling get pod -l app=harbor and kubectl -n tooling get hr harbor. (reversible=true)

## Unresolved

- Which of the two existing access keys on the IAM user is safe to delete (requires AWS admin judgement — cannot be determined from in-cluster tooling).

## Citations

[1] AWS IAM quota state (AccessKeysPerUser: 2) — not a GitOps change; the cluster had no Git changes to the tooling/harbor workload

