---
type: Incident
title: 'HelmRelease headlamp-oauth2-proxy stuck: GCP Secret Manager secret missing, blocking ExternalSecret sync and pod startup'
description: 'The GCP Secret Manager secret ''headlamp-oauth2-proxy'' does not exist (or the External Secrets service account lacks IAM access to it) in project ogenki-435905. The ExternalSecret tooling/headlamp-oauth2-proxy repeatedly fails with ''PermissionDenied: Permission secretmanager.versions.access denied on resource (or it may not exist)'', so the Kubernetes Secret ''headlamp-oauth2-proxy'' is never created. The oauth2-proxy pod then fails with CreateContainerConfigError: secret ''headlamp-oauth2-proxy'' not found, the Deployment never becomes Ready, and the HelmRelease install is stuck (Ready=Unknown, Progressing, 5m timeout) and will eventually time out to InstallFailed.'
resource: tooling/headlamp-oauth2-proxy
tags:
    - runlore
    - incident
    - helmrelease
    - tooling
timestamp: "2026-08-28T15:37:57Z"
fingerprint: 76c340d5aba666e8f410c67fc1cdfdd4df9465b20928f2c3b00d770c7bad7b4a
confidence: 0.82
---

## Decision

- **why keep:** The GCP Secret Manager secret 'headlamp-oauth2-proxy' does not exist (or the External Secrets service account lacks IAM access to it) in project ogenki-435905. The ExternalSecret tooling/headlamp-oauth2-proxy repeatedly fails with 'PermissionDenied: Permission secretmanager.versions.access denied on resource (or it may not exist)', so the Kubernetes Secret 'headlamp-oauth2-proxy' is never created. The oauth2-proxy pod then fails with CreateContainerConfigError: secret 'headlamp-oauth2-proxy' not found, the Deployment never becomes Ready, and the HelmRelease install is stuck (Ready=Unknown, Progressing, 5m timeout) and will eventually time out to InstallFailed.
- **confidence:** 82%

## Symptom

HelmRelease headlamp-oauth2-proxy stuck: GCP Secret Manager secret missing, blocking ExternalSecret sync and pod startup

Affected resource: HelmRelease tooling/headlamp-oauth2-proxy

## Investigate

- kube_events (tooling, object=headlamp-oauth2-proxy): 8+ Warning events from 15:30:22Z to 15:34:38Z — 'UpdateFailed: error processing spec.dataFrom[0].extract, err: unable to access Secret from SecretManager Client: rpc error: code = PermissionDenied desc = Permission secretmanager.versions.access denied on resource (or it may not exist)'
- pod_status (tooling, selector app=oauth2-proxy): 'headlamp-oauth2-proxy-57d588f5f5-h8th4 Pending ready=0/1 — oauth2-proxy: CreateContainerConfigError: secret headlamp-oauth2-proxy not found'
- resource_spec ExternalSecret tooling/headlamp-oauth2-proxy: status.conditions Ready=False reason=SecretSyncedError message='could not get secret data from provider'; spec.dataFrom[0].extract.key='headlamp-oauth2-proxy', secretStoreRef=ClusterSecretStore/clustersecretstore
- resource_spec ExternalSecret tooling/headlamp-envvars (SAME ClusterSecretStore): Ready=True reason=SecretSynced message='secret synced' — proves the store and its GCP auth work; only the headlamp-oauth2-proxy GCP secret is missing/inaccessible
- resource_spec ExternalSecret tooling/admin-password (SAME ClusterSecretStore): Ready=True reason=SecretSynced — second confirmation that the store is healthy
- gitops_resource_status HelmRelease tooling/headlamp-oauth2-proxy: Ready=Unknown (Progressing) 'Running install action with timeout of 5m0s' — the install is waiting for the Deployment to become Ready, which it can't without the Secret

## Cause

1. **The GCP Secret Manager secret 'headlamp-oauth2-proxy' does not exist (or the External Secrets service account lacks IAM access to it) in project ogenki-435905. The ExternalSecret tooling/headlamp-oauth2-proxy repeatedly fails with 'PermissionDenied: Permission secretmanager.versions.access denied on resource (or it may not exist)', so the Kubernetes Secret 'headlamp-oauth2-proxy' is never created. The oauth2-proxy pod then fails with CreateContainerConfigError: secret 'headlamp-oauth2-proxy' not found, the Deployment never becomes Ready, and the HelmRelease install is stuck (Ready=Unknown, Progressing, 5m timeout) and will eventually time out to InstallFailed.** (82%)

## Resolution

- Create the GCP Secret Manager secret 'headlamp-oauth2-proxy' in project ogenki-435905 with the required oauth2-proxy configuration (session secret, client ID, client secret, etc.), OR grant the External Secrets service account the 'roles/secretmanager.secretAccessor' IAM role on that secret if it already exists. Once the GCP secret is accessible, the ExternalSecret will sync within its 20m refresh interval (or force-reconcile it), the Kubernetes Secret will be created, and the oauth2-proxy pod will start. The HelmRelease should then complete its install automatically. If the HelmRelease has already timed out to InstallFailed by then, run: flux reconcile hr headlamp-oauth2-proxy -n tooling --force (reversible=true)

## Unresolved

- Whether the GCP Secret Manager secret 'headlamp-oauth2-proxy' was never created (most likely — this is a new deployment at generation 1) or was recently deleted/locked down — the GCP Secret Manager API is not directly queryable from these tools, so a human should verify with: gcloud secrets describe headlamp-oauth2-proxy --project=ogenki-435905

