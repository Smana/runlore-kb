---
type: Incident
title: '[PRE-EXISTING] SecretStore/agents-secrets Failed — OpenBao JWT auth role "agents-secrets" still not created…'
description: 'The OpenBao JWT auth role "agents-secrets" has still not been created in the external OpenBao instance (bao.priv.aws.ogenki.io, accessed via ExternalName Service security/openbao). The SecretStore/agent-system/agents-secrets validates itself by authenticating to OpenBao''s JWT auth endpoint (PUT /v1/auth/jwt/aws-0/login with role "agents-secrets"), and OpenBao returns HTTP 400: "role ''agents-secrets'' could not be found". This causes the SecretStore to be in InvalidProviderConfig/Failed status since 2026-09-26T23:28:00Z, which in turn causes the agent-secrets Kustomization''s health check to fail. The upstream GitOps dependency chain is fully healthy (security-openbao Ready=True, security Ready=True); the security-openbao Kustomization''s inventory only creates Kubernetes-side resources (ExternalName Service, ClusterIssuer, ClusterSecretStore, CA ExternalSecret) but does NOT create OpenBao-side JWT auth roles. The role must be created directly in OpenBao via its API or an external configuration mechanism outside the Flux/GitOps loop.'
resource: agent-system/agents-secrets
alert_resource: flux-system/agent-secrets
tags:
    - runlore
    - incident
    - secretstore
    - agent-system
timestamp: "2026-09-27T02:03:05Z"
fingerprint: 5213cc6afd292999cc14def1fbb7b6f2905b35aecb9b7d632f347c28a43d346f
confidence: 0.88
---

## Decision

- **why keep:** The OpenBao JWT auth role "agents-secrets" has still not been created in the external OpenBao instance (bao.priv.aws.ogenki.io, accessed via ExternalName Service security/openbao). The SecretStore/agent-system/agents-secrets validates itself by authenticating to OpenBao's JWT auth endpoint (PUT /v1/auth/jwt/aws-0/login with role "agents-secrets"), and OpenBao returns HTTP 400: "role 'agents-secrets' could not be found". This causes the SecretStore to be in InvalidProviderConfig/Failed status since 2026-09-26T23:28:00Z, which in turn causes the agent-secrets Kustomization's health check to fail. The upstream GitOps dependency chain is fully healthy (security-openbao Ready=True, security Ready=True); the security-openbao Kustomization's inventory only creates Kubernetes-side resources (ExternalName Service, ClusterIssuer, ClusterSecretStore, CA ExternalSecret) but does NOT create OpenBao-side JWT auth roles. The role must be created directly in OpenBao via its API or an external configuration mechanism outside the Flux/GitOps loop.
- **confidence:** 88%

## Symptom

[PRE-EXISTING] SecretStore/agents-secrets Failed — OpenBao JWT auth role "agents-secrets" still not created (occurrence #4)

Affected resource: SecretStore agent-system/agents-secrets

## Investigate

- kube_events (agent-system): Warning SecretStore/agents-secrets InvalidProviderConfig (x28) — 'Error making API request. URL: PUT https://openbao.security.svc.cluster.local:8200/v1/auth/jwt/aws-0/login Code: 400. Errors: * role "agents-secrets" could not be found' (last seen 2026-09-27T01:56:31Z)
- resource_spec (SecretStore/agent-system/agents-secrets): status.conditions[0] type=Ready status=False reason=InvalidProviderConfig message='unable to create client' lastTransitionTime=2026-09-26T23:28:00Z; spec.provider.vault.auth.jwt.role='agents-secrets' path='jwt/aws-0' server='https://openbao.security.svc.cluster.local:8200'
- gitops_resource_status (Kustomization/flux-system/agent-secrets): Ready=False (HealthCheckFailed) — 'health check failed after 32.852508ms: failed early due to stalled resources: [SecretStore/agent-system/agents-secrets status: Failed]'
- gitops_tree (Kustomization/agent-secrets): ALL upstream dependencies Ready=True — security-openbao Ready=True, security Ready=True, eks-pod-identities Ready=True, full chain healthy; the bottleneck is the SecretStore itself, not a dependency
- resource_spec (Kustomization/security-openbao): status Ready=True Healthy=True; inventory contains only Service/security_openbao, ClusterIssuer/openbao, ClusterSecretStore (3x), ExternalSecret/security_openbao-ca — NO OpenBao JWT auth role resources
- controller_logs (kustomize-controller, agent-secrets): server-side apply completes successfully every 30s ('SecretStore/agent-system/agents-secrets: unchanged', 'ServiceAccount/agent-system/agents-secrets: unchanged', 'ExternalSecret/agent-system/openbao-ca: unchanged') but health check fails immediately after with 'failed early due to stalled resources: [SecretStore/agent-system/agents-secrets status: Failed]'
- logs_error_summary (external-secrets controller, security ns): 'unable to validate store' (x8, first 01:07:31Z → last 01:56:31Z) and 'Reconciler error' (x8, same span) — the external-secrets controller is failing to validate the SecretStore on every reconciliation
- resource_spec (Service/security/openbao): type=ExternalName externalName='bao.priv.aws.ogenki.io' port=8200 — OpenBao is an external managed service, not a pod in the cluster

## Cause

1. **The OpenBao JWT auth role "agents-secrets" has still not been created in the external OpenBao instance (bao.priv.aws.ogenki.io, accessed via ExternalName Service security/openbao). The SecretStore/agent-system/agents-secrets validates itself by authenticating to OpenBao's JWT auth endpoint (PUT /v1/auth/jwt/aws-0/login with role "agents-secrets"), and OpenBao returns HTTP 400: "role 'agents-secrets' could not be found". This causes the SecretStore to be in InvalidProviderConfig/Failed status since 2026-09-26T23:28:00Z, which in turn causes the agent-secrets Kustomization's health check to fail. The upstream GitOps dependency chain is fully healthy (security-openbao Ready=True, security Ready=True); the security-openbao Kustomization's inventory only creates Kubernetes-side resources (ExternalName Service, ClusterIssuer, ClusterSecretStore, CA ExternalSecret) but does NOT create OpenBao-side JWT auth roles. The role must be created directly in OpenBao via its API or an external configuration mechanism outside the Flux/GitOps loop.** (88%)

## Resolution

- Create the JWT auth role 'agents-secrets' in OpenBao at the auth/jwt/aws-0 mount path. This requires direct OpenBao API access (e.g., 'bao write auth/jwt/aws-0/role/agents-secrets role_type=jwt bound_audiences=openbao user_claim=sub policies=<appropriate-policy> token_ttl=<ttl>') or whatever external mechanism provisions OpenBao roles for this platform. Once the role exists, the SecretStore will validate on the next external-secrets reconciliation cycle (~1m), the health check will pass, and the agent-secrets Kustomization will become Ready. (reversible=true)

## Unresolved

- What mechanism is supposed to create OpenBao JWT auth roles for this platform? The security-openbao Kustomization does not include role creation in its inventory, and no Crossplane managed resource or Job for OpenBao role provisioning was found. The role must be created out-of-band (manually or via an external automation tool), and determining the intended provisioning workflow requires human knowledge of the platform's OpenBao setup.

