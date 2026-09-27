---
type: Incident
title: Kustomization/security-openbao ReconciliationFailed — transient cert-manager-webhook unavailability from spot node…
description: Spot instance terminations removed cluster nodes, causing the cert-manager-webhook pod (and other critical pods) to become unschedulable due to Insufficient cpu/memory and untolerated taints on the remaining nodes. With cert-manager-webhook down, its validating webhook service had no endpoints, causing the security-openbao Kustomization's ClusterIssuer dry-run to fail with 'no endpoints available for service cert-manager-webhook'. The incident self-healed once Karpenter provisioned replacement nodes and the webhook pod became Ready.
resource: flux-system/security-openbao
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-09-26T23:59:59Z"
fingerprint: b34738bfe447399cea6b8963ff8b0110173f3a29ff68d2ab9e68b2786479698f
confidence: 0.78
provenance:
    - 'cloud_resource_health: Karpenter nodepool default: instances=8 spot=8 terminated=3 spot-term-reasons=[Server.SpotInstanceTermination]; query_metrics_range count(kube_node_info): first=7 last=8 min=6 biggest jump +3@23:50'
---

## Decision

- **why keep:** Spot instance terminations removed cluster nodes, causing the cert-manager-webhook pod (and other critical pods) to become unschedulable due to Insufficient cpu/memory and untolerated taints on the remaining nodes. With cert-manager-webhook down, its validating webhook service had no endpoints, causing the security-openbao Kustomization's ClusterIssuer dry-run to fail with 'no endpoints available for service cert-manager-webhook'. The incident self-healed once Karpenter provisioned replacement nodes and the webhook pod became Ready.
- **confidence:** 78%
- **provenance:** cloud_resource_health: Karpenter nodepool default: instances=8 spot=8 terminated=3 spot-term-reasons=[Server.SpotInstanceTermination]; query_metrics_range count(kube_node_info): first=7 last=8 min=6 biggest jump +3@23:50

## Symptom

Kustomization/security-openbao ReconciliationFailed — transient cert-manager-webhook unavailability from spot node termination + Cilium IPAM exhaustion (self-healed)

Affected resource: Kustomization flux-system/security-openbao

## Investigate

- kube_events(security): 23:46:10 Warning Pod/cert-manager-webhook-7bc94bf9cd-h9sd6 FailedScheduling (x25): 0/7 nodes are available: 1 Insufficient memory, 2 Insufficient cpu, 4 node(s) had untolerated taint(s).
- cloud_resource_health: Karpenter nodepool default: instances=8 spot=8 terminated=3 spot-term-reasons=[Server.SpotInstanceTermination]
- query_metrics_range count(kube_node_info): first=7 last=8 min=6@21:21 biggest jump +3@23:50 — node count dropped then Karpenter provisioned replacements
- gitops_resource_status(Kustomization/security-openbao): ReconciliationFailed(x2) at 23:48:25Z — 'ClusterIssuer/security/openbao dry-run failed: no endpoints available for service cert-manager-webhook'; then Ready=True ReconciliationSucceeded at 23:52:50Z
- kube_events(security): 23:46:56 Warning Pod/cert-manager-webhook-7bc94bf9cd-h9sd6 FailedCreatePodSandBox: cilium-cni failed: unable to allocate IP via local cilium agent: [POST /ipam][502] postIpamFailure 'all CIDR ranges are exhausted'
- kube_events(security): 23:47:31 Warning Pod/cert-manager-webhook-7bc94bf9cd-h9sd6 FailedCreatePodSandBox: cilium-cni failed: unable to create endpoint: [PUT /endpoint/{id}][429] putEndpointIdTooManyRequests
- kube_events(security): 23:47:32 Warning SQLInstance/xplane-zitadel ComposeResources(x5): failed calling webhook 'mcluster.cnpg.io': no endpoints available for service 'cnpg-webhook-service' — the same class of webhook-endpoint-unavailable failure hit cnpg too, confirming cluster-wide capacity/CNI impact
- kube_events(security): 23:49:10 Warning Pod/kyverno-cleanup-controller FailedCreatePodSandBox: cilium-cni failed (add): 429 putEndpointIdTooManyRequests — kyverno also affected

## Cause

1. **Spot instance terminations removed cluster nodes, causing the cert-manager-webhook pod (and other critical pods) to become unschedulable due to Insufficient cpu/memory and untolerated taints on the remaining nodes. With cert-manager-webhook down, its validating webhook service had no endpoints, causing the security-openbao Kustomization's ClusterIssuer dry-run to fail with 'no endpoints available for service cert-manager-webhook'. The incident self-healed once Karpenter provisioned replacement nodes and the webhook pod became Ready.** (78%) — change: cloud_resource_health: Karpenter nodepool default: instances=8 spot=8 terminated=3 spot-term-reasons=[Server.SpotInstanceTermination]; query_metrics_range count(kube_node_info): first=7 last=8 min=6 biggest jump +3@23:50
2. **Secondary: Cilium CNI IPAM exhaustion on newly-provisioned nodes delayed recovery. After Karpenter scheduled the cert-manager-webhook pod onto a new node at 23:46:55, the pod sandbox could not be created because Cilium reported 'all CIDR ranges are exhausted' (502) and then '429 putEndpointIdTooManyRequests'. This extended the window during which the webhook had no endpoints, affecting not just security-openbao but also cnpg-webhook (SQLInstance/xplane-zitadel ComposeResources failures) and kyverno pods.** (55%)

## Resolution

- No immediate action required — the Kustomization has self-healed (Ready=True as of 23:52:50Z). To prevent recurrence: (1) review Karpenter nodepool 'default' spot reliance — consider adding on-demand capacity or disruption budgets for critical control-plane webhooks (cert-manager, kyverno, cnpg); (2) investigate Cilium IPAM exhaustion ('all CIDR ranges are exhausted') — increase pod CIDR range or configure Cilium for larger/more CIDR pools so new nodes can allocate pod IPs during rapid scale-up. (reversible=true)
- Investigate and remediate Cilium IPAM capacity: the 'all CIDR ranges are exhausted' error indicates the pod CIDR allocated to nodes is too small or not being released fast enough during node churn. Consider increasing the cluster pod CIDR range, configuring Cilium's --enable-pod-rotation or larger per-node CIDR masks, and ensuring Cilium gracefully releases IPs when nodes terminate. (reversible=true)

## Unresolved

- The exact time and root cause of the initial spot instance terminations (whether a single AWS spot reclaim wave or cascading failures) could not be precisely determined — CloudTrail lookup timed out (context deadline exceeded) and did not return the TerminateInstances events; the Karpenter summary confirms 3 spot terminations but not their individual timestamps or which specific nodes were lost
- network_drops tool returned an i/o timeout connecting to the network-flow source — could not independently verify whether any NetworkPolicy denials contributed; however the kube_events Cilium errors already explain the network failures, so this is corroboration-only, not a blocking gap

## Citations

[1] cloud_resource_health: Karpenter nodepool default: instances=8 spot=8 terminated=3 spot-term-reasons=[Server.SpotInstanceTermination]; query_metrics_range count(kube_node_info): first=7 last=8 min=6 biggest jump +3@23:50

