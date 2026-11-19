---
type: Incident
title: '[Pre-existing] Kustomization/merge-gate-secrets ReconciliationFailed — external-secrets-webhook (single replica)…'
description: The external-secrets-webhook Deployment (security/external-secrets-webhook) runs a single replica (replicas=1) with no pod anti-affinity or topology spread constraints. When Karpenter terminated spot node ip-10-0-1-90 at ~08:28:42Z (node went NotReady at 08:29:52Z, Karpenter reports terminated=1 in both default and agents-gvisor nodepools), the webhook pod external-secrets-webhook-6b467fd6cd-v6h5p was force-evicted. The replacement pod external-secrets-webhook-6b467fd6cd-fbg9t was scheduled onto ip-10-0-47-108 but could not get a pod IP because that node had exhausted its Cilium per-node IPAM capacity — sandbox creation failed with 'all CIDR ranges are exhausted' at 08:28:43Z and 08:28:58Z. The Cilium IPAM capacity ratio on ip-10-0-47-108 spiked to +Inf at 08:28:48Z. The webhook pod eventually obtained an IP and started at 08:29:55Z (~73s later), restoring webhook endpoints. During this gap, the Flux kustomize-controller's server-side dry-run of ExternalSecret/merge-gate/openbao-ca was rejected with 'no endpoints available for service external-secrets-webhook' at 08:29:40Z. This is the same structural fault as the prior occurrence (single replica + spot node churn), with the additional detail that Cilium IPAM exhaustion on the replacement node extended the downtime window.
resource: security/external-secrets-webhook
alert_resource: flux-system/merge-gate-secrets
tags:
    - runlore
    - incident
    - deployment
    - security
timestamp: "2026-11-19T08:59:17Z"
fingerprint: 72bd27d2a01a44d15f005e6ef80a2213d3ddf7092b06a9ce129d6d469b5252a8
confidence: 0.82
origin: runlore
---

## Decision

- **why keep:** The external-secrets-webhook Deployment (security/external-secrets-webhook) runs a single replica (replicas=1) with no pod anti-affinity or topology spread constraints. When Karpenter terminated spot node ip-10-0-1-90 at ~08:28:42Z (node went NotReady at 08:29:52Z, Karpenter reports terminated=1 in both default and agents-gvisor nodepools), the webhook pod external-secrets-webhook-6b467fd6cd-v6h5p was force-evicted. The replacement pod external-secrets-webhook-6b467fd6cd-fbg9t was scheduled onto ip-10-0-47-108 but could not get a pod IP because that node had exhausted its Cilium per-node IPAM capacity — sandbox creation failed with 'all CIDR ranges are exhausted' at 08:28:43Z and 08:28:58Z. The Cilium IPAM capacity ratio on ip-10-0-47-108 spiked to +Inf at 08:28:48Z. The webhook pod eventually obtained an IP and started at 08:29:55Z (~73s later), restoring webhook endpoints. During this gap, the Flux kustomize-controller's server-side dry-run of ExternalSecret/merge-gate/openbao-ca was rejected with 'no endpoints available for service external-secrets-webhook' at 08:29:40Z. This is the same structural fault as the prior occurrence (single replica + spot node churn), with the additional detail that Cilium IPAM exhaustion on the replacement node extended the downtime window.
- **confidence:** 82%

## Symptom

[Pre-existing] Kustomization/merge-gate-secrets ReconciliationFailed — external-secrets-webhook (single replica) evicted by spot node termination, replacement pod blocked by Cilium IPAM exhaustion on target node

Affected resource: Deployment security/external-secrets-webhook

## Investigate

- pod_status: external-secrets-webhook-6b467fd6cd-fbg9t Running ready=1/1 age=21m on ip-10-0-47-108 (replacement pod); the prior pod external-secrets-webhook-6b467fd6cd-v6h5p was evicted
- kube_events: Pod/external-secrets-webhook-6b467fd6cd-v6h5p Evicted at 08:28:42Z 'Forceful Termination' and Killing: Stopping container webhook
- kube_events: Pod/external-secrets-webhook-6b467fd6cd-fbg9t FailedCreatePodSandBox at 08:28:43Z and 08:28:58Z: 'cilium-cni failed (add): unable to allocate IP via local cilium agent: [POST /ipam][502] postIpamFailure "all CIDR ranges are exhausted"'
- kube_events: Pod/external-secrets-webhook-6b467fd6cd-fbg9t Scheduled to ip-10-0-47-108 at 08:28:42Z, container Started at 08:29:55Z — ~73s gap with no webhook endpoints
- query_metrics_range: cilium_ip_addresses/cilium_ipam_capacity ratio for ip-10-0-47-108 spiked to +Inf at 08:28:48Z (biggest jump +Inf), confirming Cilium IPAM exhaustion on that node at the exact time the replacement pod was placed there
- query_metrics_range: kube_node_status_condition Ready for ip-10-0-1-90 went from 1→0 at 08:29:52Z (node terminated); cilium_operator_ipam_nodes{category="total"} dropped from 9→8 at 08:29:36Z
- cloud_resource_health: Karpenter nodepool default: instances=6 spot=6 terminated=1; Karpenter nodepool agents-gvisor: instances=2 spot=2 terminated=1 — spot node termination confirmed
- resource_spec Deployment security/external-secrets-webhook: replicas=1, no spec.template.spec.affinity or topologySpreadConstraints — single replica with no HA scheduling constraints
- gitops_resource_status: Kustomization flux-system/merge-gate-secrets Ready=True (ReconciliationSucceeded) at 08:49:34Z after having failed at 08:29:40Z with the webhook no-endpoints error — the alert has self-healed
- controller_logs kustomize-controller: at 08:54:36Z 'server-side apply completed' for merge-gate-secrets with ExternalSecret/merge-gate/openbao-ca: unchanged — reconciliation is now succeeding
- query_metrics_range: cilium_ip_addresses{family="ipv4"} / cilium_ipam_capacity for ip-10-0-47-108: first=0.7 at 06:52:48Z, then spiked to +Inf at 08:28:48Z (biggest jump +Inf), last=2.3 — the node was already at 70% IPAM utilization and went over capacity when the replacement pod was scheduled
- query_metrics: cilium_ipam_capacity - cilium_ip_addresses for ip-10-0-47-108 = -13 (negative means over-capacity), confirming the node has more IPs allocated than its capacity allows
- kube_events: FailedCreatePodSandBox 'all CIDR ranges are exhausted' on external-secrets-webhook-6b467fd6cd-fbg9t at 08:28:43Z and 08:28:58Z — the Cilium CNI could not allocate an IP for the pod sandbox
- kb_search: runbook 'HelmRelease InstallFailed: a node at Cilium ENI IPAM capacity blocks a DaemonSet pod' documents the same Cilium IPAM exhaustion pattern with the same error message, noting the ceiling is bounded by the instance type's ENI limits under ipam: eni and recommending prefix delegation as a fix

## Cause

1. **The external-secrets-webhook Deployment (security/external-secrets-webhook) runs a single replica (replicas=1) with no pod anti-affinity or topology spread constraints. When Karpenter terminated spot node ip-10-0-1-90 at ~08:28:42Z (node went NotReady at 08:29:52Z, Karpenter reports terminated=1 in both default and agents-gvisor nodepools), the webhook pod external-secrets-webhook-6b467fd6cd-v6h5p was force-evicted. The replacement pod external-secrets-webhook-6b467fd6cd-fbg9t was scheduled onto ip-10-0-47-108 but could not get a pod IP because that node had exhausted its Cilium per-node IPAM capacity — sandbox creation failed with 'all CIDR ranges are exhausted' at 08:28:43Z and 08:28:58Z. The Cilium IPAM capacity ratio on ip-10-0-47-108 spiked to +Inf at 08:28:48Z. The webhook pod eventually obtained an IP and started at 08:29:55Z (~73s later), restoring webhook endpoints. During this gap, the Flux kustomize-controller's server-side dry-run of ExternalSecret/merge-gate/openbao-ca was rejected with 'no endpoints available for service external-secrets-webhook' at 08:29:40Z. This is the same structural fault as the prior occurrence (single replica + spot node churn), with the additional detail that Cilium IPAM exhaustion on the replacement node extended the downtime window.** (82%)
2. **Node ip-10-0-47-108, the node the webhook replacement pod was scheduled onto, had exhausted its Cilium per-node IPAM capacity (cilium_ip_addresses exceeded cilium_ipam_capacity, ratio +Inf at 08:28:48Z). This is the same class of fault documented in the KB runbook 'HelmRelease InstallFailed: a node at Cilium ENI IPAM capacity blocks a DaemonSet pod' (incidents/helmrelease-installfailed-node-at-cilium-eni-ipam-capacity-blocks-a-daemonset-pod-790c3116.md). The node's IPAM pool was already near capacity (ratio 0.7 at 06:52) and the scheduling of the replacement webhook pod pushed it over the edge, causing the CNI ADD to fail with 502 'all CIDR ranges are exhausted'. This extended the webhook downtime by ~73 seconds (08:28:42Z→08:29:55Z) until Cilium retried and an IP became available.** (45%)

## Resolution

- Increase the external-secrets-webhook Deployment to replicas=2+ with pod anti-affinity (preferredDuringScheduling) or topologySpreadConstraints (maxSkew=1, topologyKey=kubernetes.io/hostname) so that a single node termination never takes the webhook to zero replicas. Additionally, the Cilium IPAM exhaustion on ip-10-0-47-108 (capacity ratio exceeded 1.0) indicates that node is over-subscribed — consider spreading pods more evenly across nodes or increasing per-node IPAM capacity (switch Cilium from ipam: eni to prefix delegation). (reversible=true)
- Monitor cilium_operator_ipam_nodes{category="at-capacity"} > 0 as an early warning. Move Cilium from ipam: eni to prefix delegation to raise per-node IPAM capacity substantially, or spread pods more evenly across nodes via anti-affinity/topology spread constraints to avoid any single node reaching IPAM ceiling. (reversible=true)

## Unresolved

- Whether the cluster should move Cilium from ipam: eni to prefix delegation is an architectural decision that depends on the cluster's ENI limits and pod density targets — a human should evaluate this against the cluster's growth trajectory.

