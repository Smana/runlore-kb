---
type: Incident
title: Kustomization/zitadel ReconciliationFailed — Karpenter spot node termination evicted cert-manager-webhook (no…
description: 'Karpenter spot node termination(s) around 09:45-09:47 evicted admission webhook pods (cert-manager-webhook, external-secrets-webhook, cnpg-webhook-service) from the security and infrastructure namespaces. With cert-manager-webhook having no available endpoints, the Kustomization/zitadel dry-run apply of Certificate/security/zitadel at 09:45:51Z failed with ''no endpoints available for service cert-manager-webhook''. The cluster was already capacity-constrained — node ip-10-0-4-96 had been NotReady since 08:37 — so the evicted webhook pods could not immediately reschedule (FailedScheduling at 09:45:32Z: ''0/7 nodes are available: 1 untolerated taint, 2 Insufficient memory, 5 Insufficient cpu''). Karpenter provisioned a replacement node (ip-10-0-37-13 at 09:48), webhook pods rescheduled and became ready, and the Kustomization self-healed at 09:47:56Z. This is the same recurrence pattern as the past karpenter-nodepools incident where node rotation evicted Kyverno and other webhooks.'
resource: flux-system/zitadel
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-11-09T09:57:32Z"
fingerprint: a63d8db4c48e8f5421057f9c0f66a58a43f0a70dd390587ac2ef215f7f02dba7
confidence: 0.85
origin: runlore
---

## Decision

- **why keep:** Karpenter spot node termination(s) around 09:45-09:47 evicted admission webhook pods (cert-manager-webhook, external-secrets-webhook, cnpg-webhook-service) from the security and infrastructure namespaces. With cert-manager-webhook having no available endpoints, the Kustomization/zitadel dry-run apply of Certificate/security/zitadel at 09:45:51Z failed with 'no endpoints available for service cert-manager-webhook'. The cluster was already capacity-constrained — node ip-10-0-4-96 had been NotReady since 08:37 — so the evicted webhook pods could not immediately reschedule (FailedScheduling at 09:45:32Z: '0/7 nodes are available: 1 untolerated taint, 2 Insufficient memory, 5 Insufficient cpu'). Karpenter provisioned a replacement node (ip-10-0-37-13 at 09:48), webhook pods rescheduled and became ready, and the Kustomization self-healed at 09:47:56Z. This is the same recurrence pattern as the past karpenter-nodepools incident where node rotation evicted Kyverno and other webhooks.
- **confidence:** 85%

## Symptom

Kustomization/zitadel ReconciliationFailed — Karpenter spot node termination evicted cert-manager-webhook (no endpoints), self-healed in ~2min

Affected resource: Kustomization flux-system/zitadel

## Investigate

- query_metrics_range kube_node_status_condition{status!='true'}: ip-10-0-15-111 NotReady max=1@09:46:00Z (recovered 09:47:00Z); ip-10-0-46-250 NotReady max=1@09:47:00Z (stayed down); ip-10-0-4-96 NotReady since 08:37:00Z
- kube_events(security): at 09:45:32Z Pod/cert-manager-webhook-57f565cf59-qf6wp FailedScheduling (x25): '0/7 nodes are available: 1 node(s) had untolerated taint(s), 2 Insufficient memory, 5 Insufficient cpu'
- kube_events(security): at 09:46:25Z SQLInstance/xplane-zitadel ComposeResources: 'no endpoints available for service external-secrets-webhook'; at 09:46:58Z: 'no endpoints available for service cnpg-webhook-service' — multiple webhooks down simultaneously confirming cluster-wide node disruption
- kube_events(flux-system/zitadel): at 09:45:51Z Warning ReconciliationFailed (x2): 'Certificate/security/zitadel dry-run failed: no endpoints available for service cert-manager-webhook'
- gitops_resource_status(Kustomization/zitadel): Ready=True (ReconciliationSucceeded) at 09:47:56Z — self-healed; controller_logs(kustomize-controller) confirm continuous successful reconciliations at 09:52-09:55Z with 'Certificate/security/zitadel: unchanged'
- pod_status(security): all webhook pods (cert-manager-webhook, cert-manager, external-secrets-webhook, external-secrets-cert-controller, kyverno-*) have age=6m at time of check (~09:51), confirming mass restart/eviction around 09:45
- cloud_resource_health: 'Karpenter nodepool default: instances=7 spot=7 terminated=1'; query_metrics_range shows new node ip-10-0-37-13 appearing at 09:48:00Z (Karpenter provisioned replacement)
- what_changed(security): no GitOps changes at incident time — only unrelated earlier Kustomization syncs (crds-external-secrets at 15:03, crds-kyverno at 22:13 previous day)
- query_metrics_range kube_node_status_condition{status!='true'}: ip-10-0-4-96 first=0 last=1 max=1@08:37:00Z — NotReady since 08:37 and stayed NotReady through the incident
- query_metrics_range kube_node_status_condition{status='true'}: ip-10-0-4-96 trend shows 1>1>1>1>1>1>1>1>1>0>0>0, dropped to 0 at 08:37:00Z and never recovered
- kube_events(security): FailedScheduling at 09:45:32Z reports '0/7 nodes are available' — with ip-10-0-4-96 already down and two more nodes going NotReady, only 7 of 10 nodes remained and none had sufficient capacity

## Cause

1. **Karpenter spot node termination(s) around 09:45-09:47 evicted admission webhook pods (cert-manager-webhook, external-secrets-webhook, cnpg-webhook-service) from the security and infrastructure namespaces. With cert-manager-webhook having no available endpoints, the Kustomization/zitadel dry-run apply of Certificate/security/zitadel at 09:45:51Z failed with 'no endpoints available for service cert-manager-webhook'. The cluster was already capacity-constrained — node ip-10-0-4-96 had been NotReady since 08:37 — so the evicted webhook pods could not immediately reschedule (FailedScheduling at 09:45:32Z: '0/7 nodes are available: 1 untolerated taint, 2 Insufficient memory, 5 Insufficient cpu'). Karpenter provisioned a replacement node (ip-10-0-37-13 at 09:48), webhook pods rescheduled and became ready, and the Kustomization self-healed at 09:47:56Z. This is the same recurrence pattern as the past karpenter-nodepools incident where node rotation evicted Kyverno and other webhooks.** (85%)
2. **Node ip-10-0-4-96 went NotReady at 08:37Z (over an hour before the incident) and did not recover, reducing cluster capacity by one node. This pre-existing capacity deficit compounded the impact of the subsequent spot node terminations at 09:46-09:47, leaving insufficient CPU/memory for webhook pods to reschedule and extending the window of webhook unavailability.** (50%)

## Resolution

- Prevent recurrence by ensuring admission webhook pods (cert-manager-webhook, external-secrets-webhook, cnpg-webhook-service) are resilient to node churn: (1) add PodDisruptionBudgets for these webhook Deployments so at least one replica survives voluntary disruptions, (2) set webhook failurePolicy to IfNotPresent where appropriate so Kubernetes doesn't block server-side apply/dry-run when webhook endpoints are temporarily unavailable, (3) schedule webhook pods with anti-affinity across nodes and prefer on-demand nodes exempt from spot rotation, (4) consider running these system-level webhooks on a dedicated on-demand nodegroup rather than Karpenter spot nodes. This is the third recorded occurrence of node-rotation-evicted-webhooks causing GitOps reconciliation failures. (reversible=true)

## Unresolved

- The specific trigger for the spot node termination(s) at 09:46-09:47 could not be determined — cloud_what_changed was truncated at 25 results and the CloudTrail events shown were SSM/Karpenter provisioning events, not the termination event itself. Karpenter reports terminated=1 but the interruption notice (spot rebalance recommendation vs. forced termination) is not visible in the available tools.
- Why node ip-10-0-4-96 went NotReady at 08:37Z and never recovered — this pre-existing capacity deficit is not explained by any cloud event in the available window. It may be a failed node that Karpenter should have replaced but didn't, or a node under maintenance.

