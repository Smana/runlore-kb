---
type: Incident
title: 'Harbor HelmRelease ValuesError: GCP Secret Manager denies access to ''harbor-oidc'' secret, starving the…'
description: The ExternalSecret 'harbor-oidc-config' cannot sync because GCP Secret Manager denies 'secretmanager.versions.access' on the 'harbor-oidc' secret. Since the ExternalSecret never creates the Kubernetes Secret 'harbor-oidc-config', the harbor HelmRelease's valuesFrom reference (key 'config-overwrite-json', targetPath 'core.configureUserSettings') cannot be resolved at chart-render time, producing the ValuesError.
resource: tooling/harbor
tags:
    - runlore
    - incident
    - helmrelease
    - tooling
timestamp: "2026-09-01T07:45:39Z"
fingerprint: e027412b5528c0d5ff7a9352b6ba51949cd937e9d7c638fd0a7b6878c884fb21
confidence: 0.85
---

## Decision

- **why keep:** The ExternalSecret 'harbor-oidc-config' cannot sync because GCP Secret Manager denies 'secretmanager.versions.access' on the 'harbor-oidc' secret. Since the ExternalSecret never creates the Kubernetes Secret 'harbor-oidc-config', the harbor HelmRelease's valuesFrom reference (key 'config-overwrite-json', targetPath 'core.configureUserSettings') cannot be resolved at chart-render time, producing the ValuesError.
- **confidence:** 85%

## Symptom

Harbor HelmRelease ValuesError: GCP Secret Manager denies access to 'harbor-oidc' secret, starving the harbor-oidc-config ExternalSecret

Affected resource: HelmRelease tooling/harbor

## Investigate

- HelmRelease/harbor status: Ready=False reason=ValuesError, message='could not resolve Secret chart values reference tooling/harbor-oidc-config with key config-overwrite-json: secrets harbor-oidc-config not found' (gitops_resource_status)
- HelmRelease spec confirms valuesFrom: kind=Secret, name=harbor-oidc-config, valuesKey=config-overwrite-json, targetPath=core.configureUserSettings (resource_spec)
- ExternalSecret/harbor-oidc-config status: Ready=False reason=SecretSyncedError message='could not get secret data from provider' (resource_spec)
- kube_events: repeated Warning UpdateFailed on ExternalSecret/harbor-oidc-config since 07:34:32Z: 'rpc error: code = PermissionDenied desc = Permission secretmanager.versions.access denied on resource (or it may not exist)' reason=IAM_PERMISSION_DENIED
- The ExternalSecret spec shows dataFrom[0].extract.key=harbor-oidc via ClusterSecretStore/clustersecretstore (gcpsm provider, projectID=ogenki-435905) (resource_spec)
- ExternalSecret/admin-password: [REDACTED], reason=SecretSynced, message='secret synced', refreshTime=07:34:30Z (resource_spec)
- ExternalSecret/harbor-valkey-password: [REDACTED], reason=SecretSynced, message='secret synced', refreshTime=07:34:33Z (resource_spec)
- ClusterSecretStore/clustersecretstore: Ready=True, reason=Valid, message='store validated' (resource_spec)
- All three ExternalSecrets use the same ClusterSecretStore/clustersecretstore and the same GCP project (ogenki-435905)

## Cause

1. **The ExternalSecret 'harbor-oidc-config' cannot sync because GCP Secret Manager denies 'secretmanager.versions.access' on the 'harbor-oidc' secret. Since the ExternalSecret never creates the Kubernetes Secret 'harbor-oidc-config', the harbor HelmRelease's valuesFrom reference (key 'config-overwrite-json', targetPath 'core.configureUserSettings') cannot be resolved at chart-render time, producing the ValuesError.** (85%)
2. **The denial is isolated to the 'harbor-oidc' GCP Secret Manager secret — not a broken ESO identity or store. The same ClusterSecretStore successfully syncs 'harbor-admin-password' and 'harbor-valkey-password', proving the GCP workload identity has Secret Manager access at the project level. This points to the 'harbor-oidc' secret either being absent from GCP Secret Manager or carrying restrictive per-secret IAM.** (50%)

## Resolution

- Check the 'harbor-oidc' secret in GCP Secret Manager (project ogenki-435905). If it does not exist, create it with the expected fields (endpoint, client_id, client_secret for the ZITADEL OIDC integration — see the ExternalSecret template). If it exists, verify its IAM grants the ESO workload identity 'roles/secretmanager.secretAccessor' on that specific secret. Once the secret is accessible, the ExternalSecret will sync within its 20m refreshInterval, creating the Kubernetes Secret, and the harbor HelmRelease will auto-retry (install.remediation.retries=-1) and resolve the ValuesError. (reversible=true)
- Run 'gcloud secrets describe harbor-oidc --project=ogenki-435905' to confirm existence. If missing, create it. If present, run 'gcloud secrets get-iam-policy harbor-oidc --project=ogenki-435905' and verify the ESO workload identity service account has 'roles/secretmanager.secretAccessor'. (reversible=true)

## Unresolved

- Whether the 'harbor-oidc' GCP Secret Manager secret was supposed to be created by an IaC/Terraform/Crossplane bootstrap step that hasn't run or failed during this cluster rebuild. The evidence suggests a fresh cluster install (all tooling pods are 2-7 minutes old, HelmRelease observedGeneration=-1), but I could not identify what mechanism is responsible for provisioning this secret in GCP Secret Manager.

