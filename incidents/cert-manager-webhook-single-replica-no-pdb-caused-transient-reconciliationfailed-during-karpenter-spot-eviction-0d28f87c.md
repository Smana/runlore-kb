---
type: Incident
title: cert-manager-webhook single-replica (no PDB) caused transient ReconciliationFailed during Karpenter spot eviction
description: 'The cert-manager-webhook Deployment in namespace security runs with replicas=1 (single replica) and has no PodDisruptionBudget. When Karpenter''s default nodepool lost 2 spot instances to Server.SpotInstanceTermination (BidEvictedEvent at 09:43:24Z), the node hosting the sole cert-manager-webhook pod was terminated. With no other replica to serve the webhook, the service cert-manager-webhook.security.svc had no endpoints. The Flux kustomize-controller''s server-side dry-run of Certificate/infrastructure/barman-cloud-client was rejected by the admission webhook at 09:41:22Z: ''no endpoints available for service cert-manager-webhook''. A new webhook pod was rescheduled at 09:41:21Z and became Ready by ~09:42Z, and the next reconcile of the Kustomization succeeded at 09:42:23Z — the incident self-healed within ~60 seconds. The same transient webhook outage also broke Kustomization/zitadel at 09:41:21Z with the identical error.'
resource: security/cert-manager-webhook
alert_resource: infrastructure/cloudnative-pg-barman-plugin
tags:
    - runlore
    - incident
    - deployment
    - security
timestamp: "2026-11-19T09:52:36Z"
fingerprint: 0d28f87c05f766b5bfccb8404907fbf9fc872bdaa0933fa4c4f696071d203411
confidence: 0.78
provenance:
    - No GitOps change — Karpenter spot eviction (Server.SpotInstanceTermination) at 2026-11-19T09:43:24Z evicted the node hosting the sole cert-manager-webhook pod
origin: runlore
---

## Decision

- **why keep:** The cert-manager-webhook Deployment in namespace security runs with replicas=1 (single replica) and has no PodDisruptionBudget. When Karpenter's default nodepool lost 2 spot instances to Server.SpotInstanceTermination (BidEvictedEvent at 09:43:24Z), the node hosting the sole cert-manager-webhook pod was terminated. With no other replica to serve the webhook, the service cert-manager-webhook.security.svc had no endpoints. The Flux kustomize-controller's server-side dry-run of Certificate/infrastructure/barman-cloud-client was rejected by the admission webhook at 09:41:22Z: 'no endpoints available for service cert-manager-webhook'. A new webhook pod was rescheduled at 09:41:21Z and became Ready by ~09:42Z, and the next reconcile of the Kustomization succeeded at 09:42:23Z — the incident self-healed within ~60 seconds. The same transient webhook outage also broke Kustomization/zitadel at 09:41:21Z with the identical error.
- **confidence:** 78%
- **provenance:** No GitOps change — Karpenter spot eviction (Server.SpotInstanceTermination) at 2026-11-19T09:43:24Z evicted the node hosting the sole cert-manager-webhook pod

## Symptom

cert-manager-webhook single-replica (no PDB) caused transient ReconciliationFailed during Karpenter spot eviction

Affected resource: Deployment security/cert-manager-webhook

## Investigate

- resource_spec Deployment/security/cert-manager-webhook: spec.replicas=1, no PDB listed in namespace security (only external-secrets-pdb, kyverno-admission-controller, xplane-zitadel-cnpg-cluster-primary exist)
- gitops_resource_status Kustomization/infrastructure/cloudnative-pg-barman-plugin: Ready=True; events show ReconciliationFailed at 09:41:22Z ('no endpoints available for service cert-manager-webhook') followed by ReconciliationSucceeded at 09:42:23Z — self-healed in ~60s
- kube_events security/cert-manager-webhook-57f565cf59-5wwkn: Scheduled at 09:41:21Z to ip-10-0-47-108, Pulling at 09:41:23Z, Started at 09:41:25Z — pod age 8m confirms recent rescheduling
- incident_timeline infrastructure: BidEvictedEvent at 09:43:24Z; cloud_resource_health reports Karpenter nodepool default: terminated=2 spot-term-reasons=[Server.SpotInstanceTermination]
- kube_events flux-system: Kustomization/zitadel ReconciliationFailed at 09:41:21Z with the identical 'no endpoints available for service cert-manager-webhook' error — same root cause, wider blast radius
- pod_status security: cert-manager-webhook-57f565cf59-5wwkn age=8m on ip-10-0-47-108; multiple other security pods (kyverno-background-controller, kyverno-cleanup-controller, external-secrets) also age=8m — all rescheduled together from the evicted node
- what_changed infrastructure: only routine revision syncs at 2026-11-09 and 2026-11-17; no GitOps change on 2026-11-19 correlates with the failure

## Cause

1. **The cert-manager-webhook Deployment in namespace security runs with replicas=1 (single replica) and has no PodDisruptionBudget. When Karpenter's default nodepool lost 2 spot instances to Server.SpotInstanceTermination (BidEvictedEvent at 09:43:24Z), the node hosting the sole cert-manager-webhook pod was terminated. With no other replica to serve the webhook, the service cert-manager-webhook.security.svc had no endpoints. The Flux kustomize-controller's server-side dry-run of Certificate/infrastructure/barman-cloud-client was rejected by the admission webhook at 09:41:22Z: 'no endpoints available for service cert-manager-webhook'. A new webhook pod was rescheduled at 09:41:21Z and became Ready by ~09:42Z, and the next reconcile of the Kustomization succeeded at 09:42:23Z — the incident self-healed within ~60 seconds. The same transient webhook outage also broke Kustomization/zitadel at 09:41:21Z with the identical error.** (78%) — change: No GitOps change — Karpenter spot eviction (Server.SpotInstanceTermination) at 2026-11-19T09:43:24Z evicted the node hosting the sole cert-manager-webhook pod

## Resolution

- Increase the cert-manager-webhook Deployment replicas from 1 to at least 2 (preferably 3) and add a PodDisruptionBudget (minAvailable: 1) in namespace security. This ensures the validating webhook always has at least one serving endpoint during node eviction/rotation. Alternatively, schedule the webhook on on-demand (non-spot) nodes using nodeAffinity or a Karpenter nodepool with capacityType=on-demand. The cert-manager Helm chart supports webhook.replicaCount and webhook.podDisruptionBudget — update the HelmRelease values in Git and reconcile. (reversible=true)

## Citations

[1] No GitOps change — Karpenter spot eviction (Server.SpotInstanceTermination) at 2026-11-19T09:43:24Z evicted the node hosting the sole cert-manager-webhook pod

