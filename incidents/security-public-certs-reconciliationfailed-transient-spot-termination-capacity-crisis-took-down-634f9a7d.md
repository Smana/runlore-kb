---
type: Incident
title: security-public-certs ReconciliationFailed — transient spot-termination capacity crisis took down…
description: 'AWS spot instance terminations (3 nodes) caused a cluster-wide capacity crunch: the cert-manager-webhook pod could not be scheduled (Insufficient cpu/memory) and, when briefly placed, could not get a pod IP (Cilium IPAM exhausted). With the webhook pod down, the cert-manager validating webhook service had no endpoints, so Flux''s server-side dry-run of ClusterIssuer/Certificate resources failed with ''no endpoints available for service cert-manager-webhook''. This affected not just security-public-certs but also zitadel and security-openbao Kustomizations. The incident has self-healed — Karpenter replaced the terminated nodes and all pods are now Running.'
resource: flux-system/security-public-certs
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-09-27T00:04:10Z"
fingerprint: 634f9a7d6b77134c301f8365af5a9127b9dd3479d1ce379da2ea4a8ac05dd844
confidence: 0.8
provenance:
    - 'No GitOps change caused this — what_changed for security-public-certs shows only a routine sync to revision bad0e0b4 at 23:56:57Z. The parent Kustomization flux-system/security is Ready=True. The trigger was infrastructure-level: AWS spot reclamation.'
---

## Decision

- **why keep:** AWS spot instance terminations (3 nodes) caused a cluster-wide capacity crunch: the cert-manager-webhook pod could not be scheduled (Insufficient cpu/memory) and, when briefly placed, could not get a pod IP (Cilium IPAM exhausted). With the webhook pod down, the cert-manager validating webhook service had no endpoints, so Flux's server-side dry-run of ClusterIssuer/Certificate resources failed with 'no endpoints available for service cert-manager-webhook'. This affected not just security-public-certs but also zitadel and security-openbao Kustomizations. The incident has self-healed — Karpenter replaced the terminated nodes and all pods are now Running.
- **confidence:** 80%
- **provenance:** No GitOps change caused this — what_changed for security-public-certs shows only a routine sync to revision bad0e0b4 at 23:56:57Z. The parent Kustomization flux-system/security is Ready=True. The trigger was infrastructure-level: AWS spot reclamation.

## Symptom

security-public-certs ReconciliationFailed — transient spot-termination capacity crisis took down cert-manager-webhook (self-healed)

Affected resource: Kustomization flux-system/security-public-certs

## Investigate

- cloud_resource_health: Karpenter nodepool 'default': instances=8 spot=8 terminated=3 spot-term-reasons=[Server.SpotInstanceTermination] — 3 spot nodes were reclaimed by AWS
- kube_events (security, 23:46:10Z): 'Pod/cert-manager-webhook-7bc94bf9cd-h9sd6 FailedScheduling (x25): 0/7 nodes are available: 1 Insufficient memory, 2 Insufficient cpu, 4 node(s) had untolerated taint(s)'
- kube_events (security, 23:46:56Z): 'Pod/cert-manager-webhook FailedCreatePodSandBox: cilium-cni failed (add): unable to allocate IP via local cilium agent: [POST /ipam][502] postIpamFailure all CIDR ranges are exhausted'
- kube_events (security, 23:47:31Z): 'Pod/cert-manager-webhook FailedCreatePodSandBox: cilium-cni failed (add): unable to create endpoint: [PUT /endpoint/{id}][429] putEndpointIdTooManyRequests'
- kube_events (flux-system, 23:47:45–23:48:35Z): Three Kustomizations failed — zitadel (Certificate dry-run), security-openbao (ClusterIssuer dry-run), security-public-certs (ClusterIssuer dry-run) — all with 'no endpoints available for service cert-manager-webhook'
- query_metrics_range (kube_node_status_condition Ready): nodes ip-10-0-25-224 and ip-10-0-31-194 went NotReady at ~23:50Z (biggest jump -1), confirming node loss; total Ready count dropped then recovered from 6→8
- query_metrics_range (sum Pending pods): peaked at 10 around 23:52Z, now 0 — capacity crunch was transient
- gitops_resource_status: Kustomization security-public-certs is now Ready=True (ReconciliationSucceeded at 23:56:57Z) — self-healed after nodes were replaced
- pod_status (security): cert-manager-webhook-7bc94bf9cd-h9sd6 now Running ready=1/1 age=14m — webhook is back online
- query_metrics (cilium_operator_ipam_nodes): at-capacity=0, total=8 — no nodes are currently IPAM-exhausted
- kube_events shows the cascade was not limited to security-public-certs: zitadel (23:47:45Z), security-openbao (23:48:25Z), and security-public-certs (23:48:35Z) all failed simultaneously with the same 'no endpoints available for service cert-manager-webhook' error
- kube_events also shows cnpg webhook failures: 'no endpoints available for service cnpg-webhook-service' blocking Crossplane SQLInstance composed resources at 23:47:31Z
- kube_events shows kyverno-cleanup-controller also FailedScheduling and FailedCreatePodSandBox with the same Cilium IPAM errors at 23:47:32Z
- KB runbook 'helmrelease-installfailed-node-at-cilium-eni-ipam-capacity' documents this exact Cilium IPAM exhaustion pattern as a known platform issue, recommending monitoring cilium_operator_ipam_nodes{category='at-capacity'}

## Cause

1. **AWS spot instance terminations (3 nodes) caused a cluster-wide capacity crunch: the cert-manager-webhook pod could not be scheduled (Insufficient cpu/memory) and, when briefly placed, could not get a pod IP (Cilium IPAM exhausted). With the webhook pod down, the cert-manager validating webhook service had no endpoints, so Flux's server-side dry-run of ClusterIssuer/Certificate resources failed with 'no endpoints available for service cert-manager-webhook'. This affected not just security-public-certs but also zitadel and security-openbao Kustomizations. The incident has self-healed — Karpenter replaced the terminated nodes and all pods are now Running.** (80%) — change: No GitOps change caused this — what_changed for security-public-certs shows only a routine sync to revision bad0e0b4 at 23:56:57Z. The parent Kustomization flux-system/security is Ready=True. The trigger was infrastructure-level: AWS spot reclamation.
2. **Systemic risk: critical Kubernetes admission webhooks (cert-manager, cnpg, kyverno) run on spot instances with insufficient redundancy. When spot nodes are reclaimed, these webhooks go offline, causing cascading GitOps reconciliation failures across multiple Kustomizations and namespaces — not just the one that alerted. This is the same pattern documented in the KB runbook for Cilium IPAM capacity blocking DaemonSet pods.** (50%)

## Resolution

- No immediate action needed — the Kustomization has self-healed. To prevent recurrence: (1) consider running critical admission webhooks (cert-manager, kyverno, cnpg) on on-demand nodes rather than spot, or add nodeAffinity to prefer on-demand for these control-plane-adjacent workloads; (2) ensure PodDisruptionBudgets and adequate replicas exist for webhooks so a single node loss doesn't remove all endpoints; (3) monitor cilium_operator_ipam_nodes{category='at-capacity'} > 0 as an early warning, per the KB runbook on Cilium ENI IPAM exhaustion. (reversible=false)
- Audit the scheduling of all admission webhook deployments (cert-manager-webhook, cnpg-webhook-service, kyverno-admission-controller) — ensure they have >=2 replicas with anti-affinity/topology spread across node pools that include on-demand capacity, so a single spot node termination cannot remove all webhook endpoints simultaneously. (reversible=false)

## Unresolved

- Whether the 3 spot instance terminations were part of a normal AWS spot reclamation cycle or triggered by a capacity event in the eu-west-3 AZ — a human may want to check AWS Spot Instance advisor for the incident window

## Citations

[1] No GitOps change caused this — what_changed for security-public-certs shows only a routine sync to revision bad0e0b4 at 23:56:57Z. The parent Kustomization flux-system/security is Ready=True. The trigger was infrastructure-level: AWS spot reclamation.

