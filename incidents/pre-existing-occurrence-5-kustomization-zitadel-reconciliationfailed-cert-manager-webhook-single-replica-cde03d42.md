---
type: Incident
title: '[Pre-existing, occurrence #5] Kustomization/zitadel ReconciliationFailed — cert-manager-webhook single-replica…'
description: 'The Deployment security/cert-manager-webhook runs as a single replica (replicas=1) with no PodDisruptionBudget. During a Karpenter spot node termination storm at ~18:48Z (agents-gvisor: 4/5 spot terminated, default: 2/7 terminated with spot-term-reasons=[Server.SpotInstanceTermination], node count dropped 10→7), the webhook pod (cert-manager-webhook-57f565cf59-xft5x) was evicted and went NotReady at 18:47Z. With zero webhook endpoints, the cert-manager validating webhook returned ''no endpoints available for service cert-manager-webhook'', causing Kustomization/zitadel''s dry-run of Certificate/security/zitadel to fail 4 times. The Kustomization self-healed once the replacement pod became Ready at 18:50Z and the next reconciliation at 18:49:50Z succeeded.'
resource: security/cert-manager-webhook
alert_resource: flux-system/zitadel
tags:
    - runlore
    - incident
    - deployment
    - security
timestamp: "2026-11-19T18:56:12Z"
fingerprint: cde03d42ab84a188aa016a467e6d63b4967322714415df417e04853c116ab876
confidence: 0.85
origin: runlore
---

## Decision

- **why keep:** The Deployment security/cert-manager-webhook runs as a single replica (replicas=1) with no PodDisruptionBudget. During a Karpenter spot node termination storm at ~18:48Z (agents-gvisor: 4/5 spot terminated, default: 2/7 terminated with spot-term-reasons=[Server.SpotInstanceTermination], node count dropped 10→7), the webhook pod (cert-manager-webhook-57f565cf59-xft5x) was evicted and went NotReady at 18:47Z. With zero webhook endpoints, the cert-manager validating webhook returned 'no endpoints available for service cert-manager-webhook', causing Kustomization/zitadel's dry-run of Certificate/security/zitadel to fail 4 times. The Kustomization self-healed once the replacement pod became Ready at 18:50Z and the next reconciliation at 18:49:50Z succeeded.
- **confidence:** 85%

## Symptom

[Pre-existing, occurrence #5] Kustomization/zitadel ReconciliationFailed — cert-manager-webhook single-replica (replicas=1, no PDB) evicted during Karpenter spot termination storm, compounded by Cilium CNI 429 rate-limiting; self-healed

Affected resource: Deployment security/cert-manager-webhook

## Investigate

- resource_spec Deployment/security/cert-manager-webhook: spec.replicas=1, strategy.rollingUpdate.maxUnavailable=25%, no PodDisruptionBudget for cert-manager-webhook exists (list_resources PodDisruptionBudget in security returned only external-secrets-pdb, kyverno-admission-controller, xplane-zitadel-cnpg-cluster-primary)
- gitops_resource_status Kustomization/zitadel: Ready=True (ReconciliationSucceeded) at 18:49:50Z; prior Warning ReconciliationFailed(x4) at 18:48:50Z 'Certificate/security/zitadel dry-run failed: no endpoints available for service cert-manager-webhook'
- query_metrics_range kube_pod_status_ready: pod xft5x condition=true was 1 until ~18:47Z then dropped; replacement pod hp925 condition=false first=1 at 18:47Z, condition=true first=0 reaching 1 at 18:50Z — ~3 min gap with zero ready webhook pods
- cloud_resource_health: Karpenter nodepool agents-gvisor instances=5 spot=5 terminated=4; default instances=7 spot=7 terminated=2 spot-term-reasons=[Server.SpotInstanceTermination]; cloud_what_changed shows BidEvictedEvent at 18:48:44Z and CreateFleet/RunInstances by Karpenter at 18:49:11Z for replacement nodes
- kube_pod_info metrics: two pods xft5x (original) and hp925 (replacement) — xft5x disappeared from metrics after 18:47Z, hp925 appeared at 18:47Z
- kube_deployment_status_replicas_available: dropped from 1 to 0 at 18:47Z (biggest jump -1), recovered to 1
- kube_events security: 'Pod/cert-manager-webhook-57f565cf59-hp925 FailedCreatePodSandBox (x2): failed to setup network for sandbox: plugin type=cilium-cni failed (add): unable to create endpoint: [PUT /endpoint/{id}][429] putEndpointIdTooManyRequests' at 18:47:23Z and 18:48:50Z
- query_metrics_range cilium_operator_ipam_nodes{category=at-capacity}: stayed at 0 throughout — this was NOT an IPAM capacity exhaustion (unlike the kb_search runbook), but a Cilium API rate-limit on endpoint creation during churn
- query_metrics_range kube_node_status_condition Ready: node count dropped from 10 to 7 (min=7 at 18:11Z) then recovered to 9 — massive concurrent rescheduling created the endpoint-creation burst that triggered Cilium 429s
- query_metrics_range kube_pod_status_ready hp925: condition=false=1 from 18:47Z, condition=true reached 1 at 18:50Z — the ~3 min delay from pod creation to Ready is consistent with the two FailedCreatePodSandBox retries before the Cilium endpoint was finally accepted

## Cause

1. **The Deployment security/cert-manager-webhook runs as a single replica (replicas=1) with no PodDisruptionBudget. During a Karpenter spot node termination storm at ~18:48Z (agents-gvisor: 4/5 spot terminated, default: 2/7 terminated with spot-term-reasons=[Server.SpotInstanceTermination], node count dropped 10→7), the webhook pod (cert-manager-webhook-57f565cf59-xft5x) was evicted and went NotReady at 18:47Z. With zero webhook endpoints, the cert-manager validating webhook returned 'no endpoints available for service cert-manager-webhook', causing Kustomization/zitadel's dry-run of Certificate/security/zitadel to fail 4 times. The Kustomization self-healed once the replacement pod became Ready at 18:50Z and the next reconciliation at 18:49:50Z succeeded.** (85%)
2. **The replacement cert-manager-webhook pod (hp925) was further delayed by ~2-3 minutes because Cilium CNI returned HTTP 429 putEndpointIdTooManyRequests when creating the pod's network endpoint. This happened during the massive endpoint churn caused by simultaneous pod rescheduling across the cluster (node count dropped 10→7, multiple Deployments rescheduling). Two FailedCreatePodSandBox events at 18:47:23Z and 18:48:50Z show the Cilium operator was rate-limiting endpoint creation, extending the window during which the webhook had zero endpoints from what would have been ~1 min to ~3 min.** (65%)

## Resolution

- Increase cert-manager-webhook Deployment replicas to ≥2 with podAntiAffinity (topologyKey: kubernetes.io/hostname) and add a PodDisruptionBudget (minAvailable: 1) so the validating webhook always has at least one endpoint during node rotation. This is the 5th recurrence of this fault — the single-replica/no-PDB pattern has now caused repeated Kustomization/zitadel ReconciliationFailed alerts during every spot node churn event. (reversible=true)
- This is a secondary contributor. If Cilium 429 rate-limiting recurs during node churn storms, consider tuning Cilium's endpoint-regeneration rate limits or ensuring Karpenter disruption budgets stagger node terminations to avoid burst rescheduling. The primary fix (root cause #1: multi-replica webhook + PDB) would make this transient delay non-impactful. (reversible=true)

