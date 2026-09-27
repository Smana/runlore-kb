---
type: Incident
title: 'octo-sts Kustomization HealthCheckFailed: OpenBao secret data missing at agents/github-app'
description: 'The OpenBao secret data at path ''agents/github-app'' (within the ''platform'' mount) does not exist, so ExternalSecret ''octo-sts-github-app'' cannot sync and Kubernetes Secret ''octo-sts-github-app'' is never created. The octo-sts pod requires this Secret for both the GITHUB_APP_IDS env var (key: app_id) and the ''app-key'' volume mount (key: private_key → private-key.pem). Without it, the pod gets FailedMount events and stays Pending, the Deployment never becomes Ready, and the Flux Kustomization health check times out after 3 minutes.'
resource: flux-system/octo-sts
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-09-27T04:47:10Z"
fingerprint: 3a3932c8a3c7e13468047a4a5f8e99763c25a553136dfbf5a8cab9647928c4ff
confidence: 0.85
---

## Decision

- **why keep:** The OpenBao secret data at path 'agents/github-app' (within the 'platform' mount) does not exist, so ExternalSecret 'octo-sts-github-app' cannot sync and Kubernetes Secret 'octo-sts-github-app' is never created. The octo-sts pod requires this Secret for both the GITHUB_APP_IDS env var (key: app_id) and the 'app-key' volume mount (key: private_key → private-key.pem). Without it, the pod gets FailedMount events and stays Pending, the Deployment never becomes Ready, and the Flux Kustomization health check times out after 3 minutes.
- **confidence:** 85%

## Symptom

octo-sts Kustomization HealthCheckFailed: OpenBao secret data missing at agents/github-app

Affected resource: Kustomization flux-system/octo-sts

## Investigate

- kube_events: 'Pod/octo-sts-848c4cb989-lnvfp FailedMount (x10): MountVolume.SetUp failed for volume "app-key" : secret "octo-sts-github-app" not found' (last 04:39:54Z)
- resource_spec ExternalSecret octo-sts-github-app: status Ready=False, reason SecretSyncedError, message 'could not get secret data from provider'; spec references remoteRef.key='agents/github-app' with properties app_id and private_key
- external-secrets controller logs: 'error processing spec.data[0] (key: agents/github-app), err: Secret does not exist' (x9, from 04:35:44Z to 04:39:57Z)
- resource_spec Deployment octo-sts: volume 'app-key' references secretName 'octo-sts-github-app' with item private_key→private-key.pem; env GITHUB_APP_IDS references secretKeyRef octo-sts-github-app/app_id; status Available=False (MinimumReplicasUnavailable) since 04:35:44Z
- pod_status: octo-sts-848c4cb989-lnvfp Pending ready=0/1 age=4m
- gitops_resource_status Kustomization octo-sts: 'Warning HealthCheckFailed health check failed after 3m0.015694139s: timeout waiting for: [Deployment/agent-system/octo-sts status: InProgress]' at 04:38:44Z
- kustomize-controller logs: 'Reconciliation failed after 3m0.239255471s ... error: health check failed after 3m0.017516616s: timeout waiting for: [Deployment/agent-system/octo-sts status: InProgress]' at 04:42:14Z
- kube_events: 'SecretStore/agents-secrets InvalidProviderConfig (x50): Error making API request. URL: PUT https://openbao.security.svc.cluster.local:8200/v1/auth/jwt/aws-0/login Code: 400. Errors: * role "agents-secrets" could not be found' starting 04:30:33Z
- external-secrets controller logs: 'unable to validate store ... error: could not get provider client: Error making API request. URL: PUT https://openbao.security.svc.cluster.local:8200/v1/auth/jwt/aws-0/login Code: 400. Errors: * role "agents-secrets" could not be found' (x3, 04:16:33Z→04:30:33Z)
- resource_spec SecretStore agents-secrets: status Ready=True, reason Valid, message 'store validated', lastTransitionTime 04:35:39Z — the auth role was created and the store validated
- agent-secrets Kustomization synced at 04:35:41Z (gitops_resource_status: ReconciliationSucceeded, Health check passed) — likely created/fixed the JWT auth role configuration in OpenBao
- After validation, the ExternalSecret error changed from auth failure to 'Secret does not exist' at agents/github-app — confirming auth works but data is absent

## Cause

1. **The OpenBao secret data at path 'agents/github-app' (within the 'platform' mount) does not exist, so ExternalSecret 'octo-sts-github-app' cannot sync and Kubernetes Secret 'octo-sts-github-app' is never created. The octo-sts pod requires this Secret for both the GITHUB_APP_IDS env var (key: app_id) and the 'app-key' volume mount (key: private_key → private-key.pem). Without it, the pod gets FailedMount events and stays Pending, the Deployment never becomes Ready, and the Flux Kustomization health check times out after 3 minutes.** (85%)
2. **The SecretStore 'agents-secrets' OpenBao JWT auth role was missing ('role agents-secrets could not be found', x50 InvalidProviderConfig events from 04:30:33Z) but was fixed at ~04:35:39Z when the store became Valid. This was a prerequisite that had to be resolved before the ExternalSecret could even attempt to read data — and once it could, it discovered the secret data itself is missing (root cause #1). The auth fix alone is insufficient.** (45%)

## Resolution

- Provision the GitHub App credentials in OpenBao at path 'platform/agents/github-app' with properties 'app_id' and 'private_key'. Also provision 'platform/agents/zai' with property 'api_key' (the agents-zai-api-key ExternalSecret fails identically). Once the data exists, the ExternalSecrets will sync automatically (refreshInterval: 1h, but a forced reconcile speeds this up), the Kubernetes Secret will be created, the pod will mount it and become Ready, and the Kustomization health check will pass. (reversible=true)
- Already resolved — the SecretStore is now Valid. No action needed for the auth role. Focus on provisioning the missing secret data (root cause #1). (reversible=true)

## Unresolved

- Why the secret data at OpenBao paths 'agents/github-app' and 'agents/zai' was never provisioned — whether this is a manual provisioning step that was missed, an automation/bootstrap job that failed, or the secrets were expected to be pre-populated by a different process. This is outside the scope of Kubernetes/GitOps tooling and requires a human to check the OpenBao administration/provisioning pipeline.
- Whether the security-openbao or agent-secrets Kustomization sync at ~04:35-04:40Z was supposed to provision the secret data (e.g., via a bootstrap Job or config) but didn't — the what_changed tool returned only sync metadata (revision + timestamp) for these Kustomizations, not the actual git diff, so I could not inspect what the sync was supposed to create.

