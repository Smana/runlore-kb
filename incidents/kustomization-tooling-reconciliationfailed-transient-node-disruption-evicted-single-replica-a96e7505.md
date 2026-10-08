---
type: Incident
title: Kustomization/tooling ReconciliationFailed — transient node disruption evicted single-replica…
description: Transient node disruption (ip-10-0-10-192 and ip-10-0-2-139 went NotReady at 04:02-04:04Z) evicted the external-secrets-webhook pod at 04:02:05Z. During the ~1-minute reschedule window (kill → schedule → image pull → initialDelaySeconds:20 → ready), the ValidatingWebhookConfiguration 'externalsecret-validate' had no available endpoints for service 'external-secrets-webhook'. The Flux Kustomization 'tooling' reconciliation at 04:02:26Z attempted a dry-run apply of ExternalSecret/tooling/admin-password, which triggered the webhook, hit 'no endpoints available', and failed with InternalError. The Kustomization self-healed at 04:18:21Z once the webhook pod was Ready.
resource: flux-system/tooling
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-10-08T04:26:29Z"
fingerprint: a96e75052bac68528bba34dd8ea163210d0690ac510faaf81f158169f850bdea
confidence: 0.75
provenance:
    - 'No Git change caused this — what_changed for tooling and security namespaces shows no recent diffs; HelmRelease external-secrets last upgraded 2026-10-07T18:17:09Z (10h prior); workload_ownership reports ''drift: none detected'''
---

## Decision

- **why keep:** Transient node disruption (ip-10-0-10-192 and ip-10-0-2-139 went NotReady at 04:02-04:04Z) evicted the external-secrets-webhook pod at 04:02:05Z. During the ~1-minute reschedule window (kill → schedule → image pull → initialDelaySeconds:20 → ready), the ValidatingWebhookConfiguration 'externalsecret-validate' had no available endpoints for service 'external-secrets-webhook'. The Flux Kustomization 'tooling' reconciliation at 04:02:26Z attempted a dry-run apply of ExternalSecret/tooling/admin-password, which triggered the webhook, hit 'no endpoints available', and failed with InternalError. The Kustomization self-healed at 04:18:21Z once the webhook pod was Ready.
- **confidence:** 75%
- **provenance:** No Git change caused this — what_changed for tooling and security namespaces shows no recent diffs; HelmRelease external-secrets last upgraded 2026-10-07T18:17:09Z (10h prior); workload_ownership reports 'drift: none detected'

## Symptom

Kustomization/tooling ReconciliationFailed — transient node disruption evicted single-replica external-secrets-webhook, leaving validating webhook with no endpoints

Affected resource: Kustomization flux-system/tooling

## Investigate

- kube_events (security, all_types): '2026-10-08T04:02:05Z Normal Pod/external-secrets-webhook-566f5f4bdc-gfgx8 Killing: Stopping container webhook' and '2026-10-08T04:02:05Z Normal ReplicaSet/external-secrets-webhook-566f5f4bdc SuccessfulCreate: Created pod: external-secrets-webhook-566f5f4bdc-h9kcm' — same ReplicaSet hash confirms eviction, not a rollout
- query_metrics_range kube_node_status_condition: node ip-10-0-10-192 status=false first=1 at 04:02, last=0 (recovered by 04:03); node ip-10-0-2-139 status=false first=0 last=1 min=0 max=1 at 04:04 — two nodes went NotReady
- query_metrics_range kube_pod_status_ready: external-secrets-webhook-566f5f4bdc-h9kcm ready=true first=0 at 04:02:30, reached 1 at 04:03:00 — ~1 minute gap with no ready webhook pod
- gitops_resource_status Kustomization flux-system/tooling: '2026-10-08T04:02:26Z Warning ReconciliationFailed ExternalSecret/tooling/admin-password dry-run failed (InternalError): ... no endpoints available for service external-secrets-webhook' followed by '2026-10-08T04:18:21Z Normal ReconciliationSucceeded' — self-healed
- resource_spec Deployment/security/external-secrets-webhook: replicas: 1, readinessProbe initialDelaySeconds: 20, strategy rollingUpdate maxUnavailable: 25% — single replica means any restart creates a zero-endpoint window
- cloud_resource_health: 'Karpenter nodepool default: instances=5 spot=5' — all spot instances, which are subject to interruption
- cloud_what_changed: Karpenter CreateFleet/RunInstances at 04:13-04:16Z all FAILED: Client.DryRunOperation (dry-run flag set) — Karpenter was probing for replacement capacity after the disruption
- query_metrics_range kube_node_status_condition: ip-10-0-10-192 NotReady at 04:02 (recovered by 04:03), ip-10-0-2-139 NotReady at 04:04 (did not recover in window) — two nodes disrupted in quick succession

## Cause

1. **Transient node disruption (ip-10-0-10-192 and ip-10-0-2-139 went NotReady at 04:02-04:04Z) evicted the external-secrets-webhook pod at 04:02:05Z. During the ~1-minute reschedule window (kill → schedule → image pull → initialDelaySeconds:20 → ready), the ValidatingWebhookConfiguration 'externalsecret-validate' had no available endpoints for service 'external-secrets-webhook'. The Flux Kustomization 'tooling' reconciliation at 04:02:26Z attempted a dry-run apply of ExternalSecret/tooling/admin-password, which triggered the webhook, hit 'no endpoints available', and failed with InternalError. The Kustomization self-healed at 04:18:21Z once the webhook pod was Ready.** (75%) — change: No Git change caused this — what_changed for tooling and security namespaces shows no recent diffs; HelmRelease external-secrets last upgraded 2026-10-07T18:17:09Z (10h prior); workload_ownership reports 'drift: none detected'
2. **Underlying cause of the node NotReady events is not fully determined from available data. The cluster runs Karpenter with 5 spot instances. CloudTrail shows Karpenter DryRun CreateFleet/RunInstances operations at 04:13-04:16Z (after the node disruption), consistent with Karpenter responding to capacity loss by attempting to provision replacements. This pattern is consistent with a spot instance interruption or Karpenter consolidation event, but the exact trigger (spot interruption notice vs. consolidation vs. kubelet issue) could not be confirmed from the available tooling.** (25%)

## Resolution

- Increase external-secrets-webhook replicaCount from 1 to 2+ in the HelmRelease values, and add topology spread constraints / anti-affinity so the replicas land on different nodes. This prevents any single node disruption or pod restart from taking the validating webhook offline and breaking GitOps reconciliation across the cluster. The PodDisruptionBudget (minAvailable: 1) already exists but only protects against voluntary disruptions, not node failure with a single replica. (reversible=true)
- If spot interruptions are frequent, consider adding on-demand capacity for critical control-plane components like admission webhooks, or configure Karpenter to use capacity-optimized spot allocation. Alternatively, ensure the webhook has multiple replicas with anti-affinity so a single node loss never takes it offline. (reversible=false)

## Unresolved

- Exact trigger for the node NotReady events (spot interruption vs. Karpenter consolidation vs. kubelet issue) — CloudTrail was capped at 25 events and showed only Karpenter dry-run operations after the disruption; the actual EC2 termination or interruption event was not captured
- Whether node ip-10-0-2-139 recovered after the metrics window — it was still NotReady at the end of the 120-minute range query

## Citations

[1] No Git change caused this — what_changed for tooling and security namespaces shows no recent diffs; HelmRelease external-secrets last upgraded 2026-10-07T18:17:09Z (10h prior); workload_ownership reports 'drift: none detected'

