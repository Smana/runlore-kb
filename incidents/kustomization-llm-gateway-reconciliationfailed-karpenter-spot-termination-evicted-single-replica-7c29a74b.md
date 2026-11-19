---
type: Incident
title: Kustomization/llm-gateway ReconciliationFailed — Karpenter spot termination evicted single-replica…
description: 'Karpenter spot-instance termination (nodepool ''default'': terminated=1, spot-term-reasons=[Server.SpotInstanceTermination]; nodepool ''agents-gvisor'': terminated=2) evicted the single replica of Deployment/security/external-secrets-webhook (spec.replicas=1, no PodDisruptionBudget). With no webhook pods running, Service/security/external-secrets-webhook had no endpoints, so the ''validate.externalsecret.external-secrets.io'' ValidatingWebhookConfiguration rejected all ExternalSecret create/update operations — including the dry-run Flux performs when reconciling Kustomization/flux-system/llm-gateway, which applies ExternalSecret/llm-gateway/zai-api-key. The failure timestamp (18:12:11Z) falls squarely within the node-churn window (node ip-10-0-40-177 NotReady at 17:55:35Z, new instance i-02217537f4b423082 registered at 18:11:37Z, ip-10-0-8-238 NotReady at 18:14:35Z). This is a recurrence of the same pattern documented in the KB past incident (karpenter-nodepools ArtifactFailed), where node rotation evicted admission-webhook pods and caused ''no endpoints available'' failures.'
resource: security/external-secrets-webhook
alert_resource: flux-system/llm-gateway
tags:
    - runlore
    - incident
    - deployment
    - security
timestamp: "2026-11-08T18:18:35Z"
fingerprint: 7c29a74ba880999900e05fd655e25199efdfa707650cedef2d7d031baaee263c
confidence: 0.8
provenance:
    - No GitOps change; Karpenter spot-instance termination (Server.SpotInstanceTermination) in nodepool 'default' at ~18:11Z
origin: runlore
---

## Decision

- **why keep:** Karpenter spot-instance termination (nodepool 'default': terminated=1, spot-term-reasons=[Server.SpotInstanceTermination]; nodepool 'agents-gvisor': terminated=2) evicted the single replica of Deployment/security/external-secrets-webhook (spec.replicas=1, no PodDisruptionBudget). With no webhook pods running, Service/security/external-secrets-webhook had no endpoints, so the 'validate.externalsecret.external-secrets.io' ValidatingWebhookConfiguration rejected all ExternalSecret create/update operations — including the dry-run Flux performs when reconciling Kustomization/flux-system/llm-gateway, which applies ExternalSecret/llm-gateway/zai-api-key. The failure timestamp (18:12:11Z) falls squarely within the node-churn window (node ip-10-0-40-177 NotReady at 17:55:35Z, new instance i-02217537f4b423082 registered at 18:11:37Z, ip-10-0-8-238 NotReady at 18:14:35Z). This is a recurrence of the same pattern documented in the KB past incident (karpenter-nodepools ArtifactFailed), where node rotation evicted admission-webhook pods and caused 'no endpoints available' failures.
- **confidence:** 80%
- **provenance:** No GitOps change; Karpenter spot-instance termination (Server.SpotInstanceTermination) in nodepool 'default' at ~18:11Z

## Symptom

Kustomization/llm-gateway ReconciliationFailed — Karpenter spot termination evicted single-replica external-secrets-webhook, blocking ExternalSecret validation

Affected resource: Deployment security/external-secrets-webhook

## Investigate

- gitops_resource_status Kustomization/flux-system/llm-gateway: Ready=False (ReconciliationFailed), message 'ExternalSecret/llm-gateway/zai-api-key dry-run failed (InternalError): ... no endpoints available for service external-secrets-webhook', failure event at 2026-11-08T18:12:11Z; prior success at 18:06:55Z (x385)
- resource_spec Deployment/security/external-secrets-webhook: spec.replicas=1 (single replica, no HA), no PDB referenced; status shows availableReplicas=1 only after recovery at 18:13:59Z
- cloud_resource_health: Karpenter nodepool 'default' instances=6 spot=6 terminated=1 spot-term-reasons=[Server.SpotInstanceTermination]; nodepool 'agents-gvisor' instances=2 spot=2 terminated=2
- cloud_what_changed: new EC2 instance i-02217537f4b423082 registered at 18:11:37Z (CreateTags by karpenter), volume detach at 18:11:36Z, volume attach at 18:13:11Z — node replacement activity in the 18:11-18:13 window
- query_metrics_range kube_node_status_condition: node ip-10-0-40-177 went NotReady (Ready=0) at 17:55:35Z; node ip-10-0-8-238 went NotReady at 18:14:35Z — two nodes churned in the incident window
- pod_status security: external-secrets-webhook-6cb4f7cc47-k2p86 Running ready=1/1 age=3m (rescheduled onto ip-10-0-34-228); cert-manager, kyverno, zitadel pods also age=3m — cluster-wide pod rescheduling from node churn
- kube_events security: simultaneous 'no endpoints available for service cnpg-webhook-service' at 18:12:49-18:12:50Z affecting SQLInstance/xplane-zitadel — confirms cluster-wide webhook outage from node rotation, not an external-secrets-specific issue
- kb_search returned past incident 'karpenter-nodepools ArtifactFailed' documenting the identical pattern: EKS nodegroup rotation evicted webhook pods causing 'no endpoints available' failures; resolution recommended PDBs and failurePolicy: IfNotPresent for admission webhooks
- gitops_resource_status Kustomization/flux-system/llm-gateway: still Ready=False as of investigation, last event ReconciliationFailed at 18:12:11Z, no subsequent ReconciliationSucceeded event
- pod_status security: external-secrets-webhook-6cb4f7cc47-k2p86 Running ready=1/1 age=3m — webhook is back up
- resource_spec Deployment/security/external-secrets-webhook: status.conditions Available=True at 2026-11-08T18:13:59Z, readyReplicas=1 — webhook service endpoints restored

## Cause

1. **Karpenter spot-instance termination (nodepool 'default': terminated=1, spot-term-reasons=[Server.SpotInstanceTermination]; nodepool 'agents-gvisor': terminated=2) evicted the single replica of Deployment/security/external-secrets-webhook (spec.replicas=1, no PodDisruptionBudget). With no webhook pods running, Service/security/external-secrets-webhook had no endpoints, so the 'validate.externalsecret.external-secrets.io' ValidatingWebhookConfiguration rejected all ExternalSecret create/update operations — including the dry-run Flux performs when reconciling Kustomization/flux-system/llm-gateway, which applies ExternalSecret/llm-gateway/zai-api-key. The failure timestamp (18:12:11Z) falls squarely within the node-churn window (node ip-10-0-40-177 NotReady at 17:55:35Z, new instance i-02217537f4b423082 registered at 18:11:37Z, ip-10-0-8-238 NotReady at 18:14:35Z). This is a recurrence of the same pattern documented in the KB past incident (karpenter-nodepools ArtifactFailed), where node rotation evicted admission-webhook pods and caused 'no endpoints available' failures.** (80%) — change: No GitOps change; Karpenter spot-instance termination (Server.SpotInstanceTermination) in nodepool 'default' at ~18:11Z
2. **The llm-gateway Kustomization remains in Ready=False (ReconciliationFailed) state even though the webhook has recovered (external-secrets-webhook pod ready=1/1, Deployment Available=True since 18:13:59Z). Flux reconciles on a 5-minute interval; the last attempt failed at 18:12:11Z and the next automatic reconciliation (~18:17:11Z) should succeed. Until then, the alert continues to fire and any downstream Kustomizations depending on llm-gateway remain blocked.** (50%)

## Resolution

- Immediate: trigger `flux reconcile kustomization llm-gateway -n flux-system` to clear the failure now that the webhook is back (pod ready=1/1 since 18:13:59Z). Preventive: scale Deployment/security/external-secrets-webhook to replicas>=2 with a PodDisruptionBudget (maxUnavailable: 1) and topologySpreadConstraints across zones, so a single node termination cannot take the webhook offline. Apply the same to other single-replica admission webhooks (cnpg-webhook in infrastructure was also down in this incident). Consider setting failurePolicy: IfNotPresent on the validate.externalsecret.external-secrets.io ValidatingWebhookConfiguration so Flux dry-runs are not blocked during transient webhook outages. (reversible=true)
- Run `flux reconcile kustomization llm-gateway -n flux-system` to trigger an immediate reconciliation now that the webhook endpoints are restored, rather than waiting for the next 5-minute cycle. Verify Ready=True afterward. (reversible=true)

## Unresolved

- The exact Karpenter disruption event (spot interruption notice vs. consolidation) that triggered the node termination could not be pinpointed — cloud_resource_health reports 'Server.SpotInstanceTermination' as the spot-term-reason, but the specific spot interruption notice timestamp was not available via the tools

## Citations

[1] No GitOps change; Karpenter spot-instance termination (Server.SpotInstanceTermination) in nodepool 'default' at ~18:11Z

