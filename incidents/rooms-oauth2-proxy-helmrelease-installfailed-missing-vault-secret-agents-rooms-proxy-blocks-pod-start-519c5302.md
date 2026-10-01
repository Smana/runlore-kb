---
type: Incident
title: rooms-oauth2-proxy HelmRelease InstallFailed — missing Vault secret `agents/rooms-proxy` blocks pod start
description: 'The rooms-oauth2-proxy Deployment pod cannot start because Secret `rooms-proxy` is missing, causing a CreateContainerConfigError that blocks the Deployment from becoming Ready. Helm''s --wait times out on the InProgress Deployment, so the HelmRelease install fails (InstallFailed) and is retried/uninstalled in a loop. The alert''s ''SourceNotReady'' label is a misnomer for this state: the HelmChart source itself is actually Ready (ChartPullSucceeded at 06:04:18), and the ''latest generation not reconciled'' was a brief transient at the very first reconcile (06:04:17, cleared by 06:04:19); the real fault is the install timeout from the missing Secret.'
resource: agent-system/rooms-oauth2-proxy
tags:
    - runlore
    - incident
    - helmrelease
    - agent-system
timestamp: "2026-10-01T06:13:42Z"
fingerprint: 519c530258127fa2f3bb8f119872de195efd0340599dfb41b3206531dde9bda3
confidence: 0.85
provenance:
    - no GitOps diff touched rooms-oauth2-proxy or the ExternalSecret — what_changed for agent-system returned only an unrelated xplane-rooms atlas migration configmap; the HelmRelease status shows observedGeneration=1 / firstAttemptedGeneration=1, i.e. a first-time install whose backing secret was never provisioned
---

## Decision

- **why keep:** The rooms-oauth2-proxy Deployment pod cannot start because Secret `rooms-proxy` is missing, causing a CreateContainerConfigError that blocks the Deployment from becoming Ready. Helm's --wait times out on the InProgress Deployment, so the HelmRelease install fails (InstallFailed) and is retried/uninstalled in a loop. The alert's 'SourceNotReady' label is a misnomer for this state: the HelmChart source itself is actually Ready (ChartPullSucceeded at 06:04:18), and the 'latest generation not reconciled' was a brief transient at the very first reconcile (06:04:17, cleared by 06:04:19); the real fault is the install timeout from the missing Secret.
- **confidence:** 85%
- **provenance:** no GitOps diff touched rooms-oauth2-proxy or the ExternalSecret — what_changed for agent-system returned only an unrelated xplane-rooms atlas migration configmap; the HelmRelease status shows observedGeneration=1 / firstAttemptedGeneration=1, i.e. a first-time install whose backing secret was never provisioned

## Symptom

rooms-oauth2-proxy HelmRelease InstallFailed — missing Vault secret `agents/rooms-proxy` blocks pod start

Affected resource: HelmRelease agent-system/rooms-oauth2-proxy

## Investigate

- pod_status (agent-system, app.kubernetes.io/name=oauth2-proxy): rooms-oauth2-proxy-78d6f78c76-jrjj9 Pending ready=0/1 — 'oauth2-proxy: CreateContainerConfigError: secret "rooms-proxy" not found'
- kube_events agent-system (since 120m): 2026-10-01T06:09:14Z Warning Pod/rooms-oauth2-proxy-...-zhf87 Failed (x24): Error: secret "rooms-proxy" not found; 2026-10-01T06:11:11Z Failed (x9) on the replacement pod
- gitops_resource_status HelmRelease agent-system/rooms-oauth2-proxy: Ready=Unknown (Progressing); InstallFailed event at 06:09:19Z — 'Helm install failed ... timeout waiting for: [Deployment/agent-system/rooms-oauth2-proxy status: InProgress]'
- helm-controller logs: install action started 06:04:19Z with 5m timeout; 'release is in a failed state' at 06:09:20Z; uninstall remediation then a fresh install retry — confirming the install/timeout/uninstall loop
- gitops_resource_status HelmChart tooling/agent-system-rooms-oauth2-proxy: Ready=True (ChartPullSucceeded), message 'pulled oauth2-proxy chart with version 10.7.0' — the source is NOT the problem
- resource_spec ExternalSecret agent-system/rooms-proxy: spec.dataFrom[0].extract.key=rooms-proxy, secretStoreRef=SecretStore/agents-secrets; status.conditions Ready=False reason=SecretSyncedError message='could not get secret data from provider'
- kube_events agent-system object=rooms-proxy: 2026-10-01T06:08:30Z Warning ExternalSecret/rooms-proxy UpdateFailed (x9): 'error processing spec.dataFrom[0].extract, err: Secret does not exist'
- resource_spec ExternalSecret agent-system/room-broker-oidc: spec.data[0].remoteRef.key=rooms-proxy (same vault key), status Ready=False reason=SecretSyncedError; kube_events: UpdateFailed (x9) 'error processing spec.data[0] (key: rooms-proxy), err: Secret does not exist'
- resource_spec SecretStore agent-system/agents-secrets: provider.vault.server=openbao.security.svc.cluster.local:8200, path=agents, version=v2; status.conditions Ready=True reason=Valid message='store validated' — store is healthy, so the failure is missing vault content

## Cause

1. **The rooms-oauth2-proxy Deployment pod cannot start because Secret `rooms-proxy` is missing, causing a CreateContainerConfigError that blocks the Deployment from becoming Ready. Helm's --wait times out on the InProgress Deployment, so the HelmRelease install fails (InstallFailed) and is retried/uninstalled in a loop. The alert's 'SourceNotReady' label is a misnomer for this state: the HelmChart source itself is actually Ready (ChartPullSucceeded at 06:04:18), and the 'latest generation not reconciled' was a brief transient at the very first reconcile (06:04:17, cleared by 06:04:19); the real fault is the install timeout from the missing Secret.** (85%) — change: no GitOps diff touched rooms-oauth2-proxy or the ExternalSecret — what_changed for agent-system returned only an unrelated xplane-rooms atlas migration configmap; the HelmRelease status shows observedGeneration=1 / firstAttemptedGeneration=1, i.e. a first-time install whose backing secret was never provisioned
2. **The root data fault is that ExternalSecret agent-system/rooms-proxy cannot fetch key `rooms-proxy` from the OpenBao-backed SecretStore `agents-secrets` (server openbao.security.svc.cluster.local:8200, mount path `agents`): the provider returns 'Secret does not exist'. The SecretStore itself is Valid/Ready and authenticated (status.conditions: Valid, 'store validated'), so this is not an auth or connectivity failure — the secret content is simply absent at that vault path. A second ExternalSecret (room-broker-oidc) reading the same key fails identically, corroborating that the vault path is empty rather than the ExternalSecret being misconfigured.** (72%)

## Resolution

- Provision the missing Vault/OpenBao secret at path agents/rooms-proxy (the key the ExternalSecret extracts via dataFrom.extract.key=rooms-proxy). Once the ExternalSecret syncs and the rooms-proxy Kubernetes Secret exists, force-reconcile the HelmRelease: `flux reconcile hr rooms-oauth2-proxy -n agent-system --force`. (reversible=true)
- Create/populate the OpenBao KV v2 secret at path agents/rooms-proxy with at least the keys the workloads consume (rooms-proxy needs the full secret for oauth2-proxy config existingSecret; room-broker-oidc reads keys client-id and project-id from it). After the ExternalSecrets refresh (20m interval) the Kubernetes Secrets will appear; a force-reconcile of the HelmRelease will then complete. (reversible=true)

## Unresolved

- Why the agents/rooms-proxy vault path is empty on a first-time install — whether it was never provisioned, was deleted, or lives under a different mount/path than the SecretStore expects. This requires access to OpenBao/Vault (outside this toolset) to confirm the path and key names.

## Citations

[1] no GitOps diff touched rooms-oauth2-proxy or the ExternalSecret — what_changed for agent-system returned only an unrelated xplane-rooms atlas migration configmap; the HelmRelease status shows observedGeneration=1 / firstAttemptedGeneration=1, i.e. a first-time install whose backing secret was never provisioned

