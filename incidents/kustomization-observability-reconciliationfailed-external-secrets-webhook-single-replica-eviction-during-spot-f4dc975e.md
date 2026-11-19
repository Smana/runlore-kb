---
type: Incident
title: Kustomization/observability ReconciliationFailed — external-secrets-webhook single-replica eviction during spot…
description: 'The external-secrets-webhook Deployment (security/external-secrets-webhook) runs with replicas: 1. During Karpenter spot node termination (cloud_resource_health: terminated=2, spot-term-reasons=[Server.SpotInstanceTermination]), the single webhook pod was evicted, leaving the external-secrets-webhook Service with zero endpoints. The validating webhook ''validate.externalsecret.external-secrets.io'' has failurePolicy: Fail (the external-secrets Helm default), so any ExternalSecret server-side dry-run during the endpoint gap returns InternalError instead of degrading gracefully. This blocked Flux kustomize-controller dry-run applies for 11+ Kustomizations cluster-wide (observability, zitadel, merge-gate-secrets, agentgateway-poc, infrastructure, flux-notifications, llm-gateway, observability-victoria-metrics-k8s-stack, flux-operator, security-openbao, agent-factory) from ~07:24:17Z to ~07:26:29Z. The observability Kustomization is the one that alerted.'
resource: security/external-secrets-webhook
alert_resource: flux-system/observability
tags:
    - runlore
    - incident
    - deployment
    - security
timestamp: "2026-11-09T07:51:46Z"
fingerprint: f4dc975ed9f4ac75eceeb24962a4b5988053090dbf9d7356f20c1871da175eb1
confidence: 0.82
origin: runlore
---

## Decision

- **why keep:** The external-secrets-webhook Deployment (security/external-secrets-webhook) runs with replicas: 1. During Karpenter spot node termination (cloud_resource_health: terminated=2, spot-term-reasons=[Server.SpotInstanceTermination]), the single webhook pod was evicted, leaving the external-secrets-webhook Service with zero endpoints. The validating webhook 'validate.externalsecret.external-secrets.io' has failurePolicy: Fail (the external-secrets Helm default), so any ExternalSecret server-side dry-run during the endpoint gap returns InternalError instead of degrading gracefully. This blocked Flux kustomize-controller dry-run applies for 11+ Kustomizations cluster-wide (observability, zitadel, merge-gate-secrets, agentgateway-poc, infrastructure, flux-notifications, llm-gateway, observability-victoria-metrics-k8s-stack, flux-operator, security-openbao, agent-factory) from ~07:24:17Z to ~07:26:29Z. The observability Kustomization is the one that alerted.
- **confidence:** 82%

## Symptom

Kustomization/observability ReconciliationFailed — external-secrets-webhook single-replica eviction during spot node termination (self-healed)

Affected resource: Deployment security/external-secrets-webhook

## Investigate

- gitops_resource_status Kustomization/observability: Ready=True (ReconciliationSucceeded x802 at 07:38:22Z) — the Kustomization has already self-healed; the failure was a transient Warning event at 07:24:36Z: 'ExternalSecret/runlore/runlore-slack dry-run failed (InternalError): ... no endpoints available for service external-secrets-webhook'
- resource_spec Deployment/security/external-secrets-webhook: spec.replicas=1, status.readyReplicas=1, status.availableReplicas=1 — single replica, no HA
- resource_spec PodDisruptionBudget/security/external-secrets-pdb: spec.minAvailable=1, status.expectedPods=1, status.disruptionsAllowed=0, status.currentHealthy=1 — PDB protects against voluntary disruption but NOT involuntary eviction from node termination
- cloud_resource_health: 'Karpenter nodepool default: instances=7 spot=7 terminated=2 spot-term-reasons=[Server.SpotInstanceTermination]' — confirms spot node terminations in the incident window
- kube_events Pod/external-secrets-webhook-6cb4f7cc47-8h9tv: Scheduled at 07:24:13Z, then FailedCreatePodSandBox at 07:24:15Z ('cilium-cni failed: unable to allocate IP via local cilium agent: [POST /ipam][502] postIpamFailure "all CIDR ranges are exhausted"') and 07:24:29Z (same), then 429 at 07:25:42Z — pod could not get an IP for ~2 minutes
- incident_timeline flux-system: ReconciliationFailed events cascaded across 11+ Kustomizations from 07:24:17Z (zitadel first) through 07:26:25Z, all with the identical 'no endpoints available for service external-secrets-webhook' error — confirming a cluster-wide webhook outage, not an observability-specific issue
- what_changed security namespace: last GitOps sync was crds-external-secrets on 2026-10-23 and crds-kyverno on 2026-10-20 — no configuration change caused this; it was a transient infrastructure event
- kube_events Pod/external-secrets-webhook-6cb4f7cc47-8h9tv at 07:24:15Z: 'FailedCreatePodSandBox: ... cilium-cni failed (add): unable to allocate IP via local cilium agent: [POST /ipam][502] postIpamFailure "all CIDR ranges are exhausted"'
- kube_events Pod/external-secrets-webhook-6cb4f7cc47-8h9tv at 07:25:42Z: 'FailedCreatePodSandBox: ... cilium-cni failed (add): unable to create endpoint: [PUT /endpoint/{id}][429] putEndpointIdTooManyRequests'
- kb_search matched runbook 'HelmRelease InstallFailed: a node at Cilium ENI IPAM capacity blocks a DaemonSet pod' — confirms this is a known recurring Cilium IPAM exhaustion pattern in this cluster under ipam: eni mode, bounded by instance ENI limits

## Cause

1. **The external-secrets-webhook Deployment (security/external-secrets-webhook) runs with replicas: 1. During Karpenter spot node termination (cloud_resource_health: terminated=2, spot-term-reasons=[Server.SpotInstanceTermination]), the single webhook pod was evicted, leaving the external-secrets-webhook Service with zero endpoints. The validating webhook 'validate.externalsecret.external-secrets.io' has failurePolicy: Fail (the external-secrets Helm default), so any ExternalSecret server-side dry-run during the endpoint gap returns InternalError instead of degrading gracefully. This blocked Flux kustomize-controller dry-run applies for 11+ Kustomizations cluster-wide (observability, zitadel, merge-gate-secrets, agentgateway-poc, infrastructure, flux-notifications, llm-gateway, observability-victoria-metrics-k8s-stack, flux-operator, security-openbao, agent-factory) from ~07:24:17Z to ~07:26:29Z. The observability Kustomization is the one that alerted.** (82%)
2. **Cilium ENI IPAM exhaustion on the target node (ip-10-0-46-250) delayed the external-secrets-webhook replacement pod from getting an IP for ~2 minutes (07:24:13Z scheduled → 07:26:06Z image pulled), extending the webhook endpoint gap and the resulting reconciliation failure window. The kube_events show 'all CIDR ranges are exhausted' (502) followed by 'putEndpointIdTooManyRequests' (429) from the Cilium CNI plugin. This is a known recurring pattern in this cluster (kb_search: 'HelmRelease InstallFailed: a node at Cilium ENI IPAM capacity blocks a DaemonSet pod').** (45%)

## Resolution

- Scale the external-secrets-webhook Deployment to replicas: 2 (or more) with a topology spread constraint / anti-affinity so at least one webhook pod survives node termination. The current PDB (minAvailable: 1, expectedPods: 1) is ineffective because it only blocks voluntary evictions — a spot node termination is involuntary and bypasses the PDB entirely. With 2+ replicas and minAvailable: 1, the PDB would actually prevent draining the last healthy pod. Additionally, consider setting the validating webhook failurePolicy to IfNotPresent (if acceptable for the security posture) so ExternalSecret applies degrade gracefully instead of hard-failing during webhook downtime. This is the same class of vulnerability as the past Kyverno webhook incident (kb_search: karpenter-nodepools ArtifactFailed, Kyverno 'no endpoints available for service kyverno-svc'). (reversible=true)
- This is a secondary contributing factor. Addressing the primary cause (webhook HA) would make the IPAM delay non-blocking. For the IPAM issue itself, consider: (a) spreading control-plane workloads off the densest nodes with anti-affinity/topology spread, (b) using larger instance types for nodes that attract singleton workloads, or (c) migrating Cilium to prefix delegation to raise the per-node IP ceiling. Monitor cilium_operator_ipam_nodes{category="at-capacity"} > 0. (reversible=true)

## Unresolved

- The exact Karpenter nodepool configuration that allowed 2 simultaneous spot terminations could not be determined — whether disruption budgets or termination grace periods are misconfigured. This requires reviewing the Karpenter NodePool/NodeClaim specs, which are outside the scope of the available GitOps tools for this investigation.

