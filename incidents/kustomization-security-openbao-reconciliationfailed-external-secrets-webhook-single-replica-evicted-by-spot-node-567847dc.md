---
type: Incident
title: Kustomization/security-openbao ReconciliationFailed — external-secrets-webhook single replica evicted by spot node…
description: 'The Deployment security/external-secrets-webhook runs with replicas: 1 on Karpenter spot nodes, so when the spot node hosting pod d5lv5 (node ip-10-0-9-1, which went NotReady at 20:52:30Z) was reclaimed, the pod was Evicted with ''Forceful Termination'' at 20:50:25Z. With no other replicas, the external-secrets-webhook Service had zero endpoints. When Kustomization/security-openbao tried to reconcile at 20:50:38Z and applied ClusterSecretStore/clustersecretstore, the validating webhook ''validate.clustersecretstore.external-secrets.io'' call failed with ''no endpoints available for service external-secrets-webhook''. The Kustomization self-healed at 20:54:09Z once replacement pod 6tkj7 started at 20:51:06Z on node ip-10-0-41-105. The Deployment spec confirms replicas: 1; the PDB security/external-secrets-pdb has minAvailable: 1, expectedPods: 1, disruptionsAllowed: 0 — but this PDB only protects against voluntary evictions, not forceful spot termination. This is the same fault pattern as occurrence #1 (which was cert-manager-webhook, a different single-replica webhook in the same namespace) — single-replica validating webhooks on spot nodes are structurally vulnerable to node rotation.'
resource: security/external-secrets-webhook
alert_resource: flux-system/security-openbao
tags:
    - runlore
    - incident
    - deployment
    - security
timestamp: "2026-11-18T21:03:30Z"
fingerprint: 567847dc8b0636350ace71c901ba35887bb3888bdb7532ba2ea0908f324f3b77
confidence: 0.8
provenance:
    - No GitOps change; Karpenter spot node termination (Server.SpotInstanceTermination) at ~20:50:25Z evicted the single external-secrets-webhook replica
origin: runlore
---

## Decision

- **why keep:** The Deployment security/external-secrets-webhook runs with replicas: 1 on Karpenter spot nodes, so when the spot node hosting pod d5lv5 (node ip-10-0-9-1, which went NotReady at 20:52:30Z) was reclaimed, the pod was Evicted with 'Forceful Termination' at 20:50:25Z. With no other replicas, the external-secrets-webhook Service had zero endpoints. When Kustomization/security-openbao tried to reconcile at 20:50:38Z and applied ClusterSecretStore/clustersecretstore, the validating webhook 'validate.clustersecretstore.external-secrets.io' call failed with 'no endpoints available for service external-secrets-webhook'. The Kustomization self-healed at 20:54:09Z once replacement pod 6tkj7 started at 20:51:06Z on node ip-10-0-41-105. The Deployment spec confirms replicas: 1; the PDB security/external-secrets-pdb has minAvailable: 1, expectedPods: 1, disruptionsAllowed: 0 — but this PDB only protects against voluntary evictions, not forceful spot termination. This is the same fault pattern as occurrence #1 (which was cert-manager-webhook, a different single-replica webhook in the same namespace) — single-replica validating webhooks on spot nodes are structurally vulnerable to node rotation.
- **confidence:** 80%
- **provenance:** No GitOps change; Karpenter spot node termination (Server.SpotInstanceTermination) at ~20:50:25Z evicted the single external-secrets-webhook replica

## Symptom

Kustomization/security-openbao ReconciliationFailed — external-secrets-webhook single replica evicted by spot node rotation (self-healed, pre-existing pattern #2)

Affected resource: Deployment security/external-secrets-webhook

## Investigate

- kube_events for external-secrets-webhook-6b467fd6cd-d5lv5: '2026-11-18T20:50:25Z Normal Evicted: Evicted pod: Forceful Termination' and '2026-11-18T20:50:25Z Normal Killing: Stopping container webhook'
- kube_events: '2026-11-18T20:50:31Z Warning Pod/external-secrets-webhook-6b467fd6cd-d5lv5 Unhealthy: Readiness probe failed: context deadline exceeded'
- gitops_resource_status Kustomization/security-openbao: Ready=True at 20:54:09Z; Warning ReconciliationFailed at 20:50:38Z: 'ClusterSecretStore/clustersecretstore dry-run failed (InternalError): ... no endpoints available for service external-secrets-webhook'; second failure at 20:51:08Z for ClusterSecretStore/openbao-apps
- resource_spec Deployment security/external-secrets-webhook: spec.replicas: 1, strategy RollingUpdate maxSurge 25% maxUnavailable 25%
- resource_spec PodDisruptionBudget security/external-secrets-pdb: spec.minAvailable: 1, status.expectedPods: 1, status.disruptionsAllowed: 0, status.currentHealthy: 1 — protects voluntary eviction only, not forceful spot termination
- kube_events for external-secrets-webhook-6b467fd6cd-6tkj7: Scheduled at 20:50:25Z, Created at 20:51:05Z, Started at 20:51:06Z — ~40s gap with zero ready webhook pods
- query_metrics_range node Ready: ip-10-0-9-1.eu-west-3.compute.internal went Ready=true→Ready=false at 20:52:30Z (the node hosting the evicted webhook pod)
- cloud_resource_health: Karpenter nodepool default: instances=7 spot=7 terminated=2 spot-term-reasons=[Server.SpotInstanceTermination]
- pod_status: external-secrets-webhook-6b467fd6cd-6tkj7 Running ready=1/1 age=7m at investigation time, confirming replacement pod came up recently

## Cause

1. **The Deployment security/external-secrets-webhook runs with replicas: 1 on Karpenter spot nodes, so when the spot node hosting pod d5lv5 (node ip-10-0-9-1, which went NotReady at 20:52:30Z) was reclaimed, the pod was Evicted with 'Forceful Termination' at 20:50:25Z. With no other replicas, the external-secrets-webhook Service had zero endpoints. When Kustomization/security-openbao tried to reconcile at 20:50:38Z and applied ClusterSecretStore/clustersecretstore, the validating webhook 'validate.clustersecretstore.external-secrets.io' call failed with 'no endpoints available for service external-secrets-webhook'. The Kustomization self-healed at 20:54:09Z once replacement pod 6tkj7 started at 20:51:06Z on node ip-10-0-41-105. The Deployment spec confirms replicas: 1; the PDB security/external-secrets-pdb has minAvailable: 1, expectedPods: 1, disruptionsAllowed: 0 — but this PDB only protects against voluntary evictions, not forceful spot termination. This is the same fault pattern as occurrence #1 (which was cert-manager-webhook, a different single-replica webhook in the same namespace) — single-replica validating webhooks on spot nodes are structurally vulnerable to node rotation.** (80%) — change: No GitOps change; Karpenter spot node termination (Server.SpotInstanceTermination) at ~20:50:25Z evicted the single external-secrets-webhook replica

## Resolution

- Scale the external-secrets-webhook Deployment to replicas: 2+ (with anti-affinity/topologySpreadConstraints to spread across nodes) so a single spot node termination never removes all webhook endpoints. The PDB (minAvailable: 1, expectedPods: 1) is ineffective for forceful spot terminations and should be updated to match the new replica count. This is the same structural fix needed for cert-manager-webhook (occurrence #1) — all single-replica validating webhooks in the security namespace (external-secrets-webhook, cert-manager-webhook) should be multi-replica with topology spread. Alternatively, schedule these critical webhooks on on-demand nodes exempt from spot rotation, or set failurePolicy: IfNotPresent on the external-secrets ValidatingWebhookConfiguration so Kustomization server-side-apply dry-runs are not blocked during webhook unavailability. (reversible=true)

## Citations

[1] No GitOps change; Karpenter spot node termination (Server.SpotInstanceTermination) at ~20:50:25Z evicted the single external-secrets-webhook replica

