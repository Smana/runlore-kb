---
type: Incident
title: Kustomization/zitadel ReconciliationFailed — external-secrets-webhook had no endpoints during Karpenter spot node…
description: 'The external-secrets-webhook Deployment (security/external-secrets-webhook) runs with replicas: 1 (single replica, no HA). During a period of widespread Karpenter spot instance terminations (at least 3 nodes terminated in the ''default'' nodepool, 1 in ''agents-gvisor''), the node hosting the webhook pod ip-10-0-2-98 went NotReady at ~16:22. The webhook pod q68qn was stuck on the terminating node for ~70 minutes before being force-evicted at 17:31:59. With only 1 replica and no healthy endpoints, the webhook Service ''external-secrets-webhook'' returned ''no endpoints available'', causing the Kustomization/zitadel server-side dry-run to fail when validating ExternalSecret/security/zitadel-envvars. The PodDisruptionBudget ''external-secrets-pdb'' (minAvailable: 1) only blocks voluntary disruptions (node drains) — spot terminations are involuntary and bypass PDBs entirely.'
resource: security/external-secrets-webhook
alert_resource: flux-system/zitadel
tags:
    - runlore
    - incident
    - deployment
    - security
timestamp: "2026-11-17T17:39:34Z"
fingerprint: 3d3fb152583325acb7327b574cd22cbeb36f01334f7635bb8a6a080a72e87747
confidence: 0.82
provenance:
    - No GitOps change; Karpenter spot instance termination at ~16:22 (BidEvictedEvent at 17:34:00, node ip-10-0-2-98 NotReady at 16:22)
origin: runlore
---

## Decision

- **why keep:** The external-secrets-webhook Deployment (security/external-secrets-webhook) runs with replicas: 1 (single replica, no HA). During a period of widespread Karpenter spot instance terminations (at least 3 nodes terminated in the 'default' nodepool, 1 in 'agents-gvisor'), the node hosting the webhook pod ip-10-0-2-98 went NotReady at ~16:22. The webhook pod q68qn was stuck on the terminating node for ~70 minutes before being force-evicted at 17:31:59. With only 1 replica and no healthy endpoints, the webhook Service 'external-secrets-webhook' returned 'no endpoints available', causing the Kustomization/zitadel server-side dry-run to fail when validating ExternalSecret/security/zitadel-envvars. The PodDisruptionBudget 'external-secrets-pdb' (minAvailable: 1) only blocks voluntary disruptions (node drains) — spot terminations are involuntary and bypass PDBs entirely.
- **confidence:** 82%
- **provenance:** No GitOps change; Karpenter spot instance termination at ~16:22 (BidEvictedEvent at 17:34:00, node ip-10-0-2-98 NotReady at 16:22)

## Symptom

Kustomization/zitadel ReconciliationFailed — external-secrets-webhook had no endpoints during Karpenter spot node churn (self-healed)

Affected resource: Deployment security/external-secrets-webhook

## Investigate

- gitops_resource_status Kustomization flux-system/zitadel: Ready=True (ReconciliationSucceeded) — self-healed. Events show ReconciliationFailed at 17:32:08Z with 'no endpoints available for service external-secrets-webhook', then ReconciliationSucceeded at 17:33:09Z
- resource_spec Deployment security/external-secrets-webhook: spec.replicas=1, strategy=RollingUpdate maxUnavailable=25%. status.readyReplicas=1, status.availableReplicas=1
- resource_spec PodDisruptionBudget security/external-secrets-pdb: spec.minAvailable=1, selector app.kubernetes.io/name=external-secrets (covers the main controller, not the webhook specifically). status.conditions DisruptionAllowed=False, disruptionsAllowed=0
- kube_events Pod/external-secrets-webhook-6b467fd6cd-q68qn: Normal Evicted at 17:31:59Z 'Evicted pod: Forceful Termination', Normal Killing 'Stopping container webhook' at 17:31:59Z
- kube_events Pod/external-secrets-webhook-6b467fd6cd-d7wdc: Normal Scheduled at 17:31:59Z to ip-10-0-39-158, Created/Started at 17:32:07Z — replacement pod
- query_metrics_range kube_pod_status_ready condition=true for external-secrets-webhook pods: q68qn went from Ready=1 to Ready=0 at 16:23 (node NotReady), d7wdc became Ready=1 at 17:33. Sum of ready replicas: 0 from 16:22 to 17:33
- query_metrics_range kube_node_status_condition: ip-10-0-2-98 NotReady at 16:22, ip-10-0-44-159 NotReady at 16:20, ip-10-0-43-91 NotReady at 16:37, ip-10-0-8-14 NotReady at 16:24, ip-10-0-30-205 NotReady at 17:22, ip-10-0-3-118 NotReady at 17:18 — widespread node churn
- cloud_resource_health: Karpenter nodepool default: instances=8 spot=8 terminated=3 spot-term-reasons=[Server.SpotInstanceTermination]; agents-gvisor: terminated=1
- cloud_what_changed: BidEvictedEvent at 17:34:00Z, DetachVolume at 17:34:09Z, AttachVolume at 17:34:57Z — node replacement in progress
- controller_logs kustomize-controller zitadel: ReconciliationSucceeded at 17:34:11Z with all resources 'unchanged' — confirming the Kustomization self-healed once the webhook was back
- pod_logs external-secrets-cert-controller: healthz check failed at 17:32:00Z (x7 through 17:32:25Z) — the cert-controller also experienced healthz failures during the webhook downtime, returning HTTP 500 on its readiness probe
- kube_events Pod/external-secrets-cert-controller-9667b79b6-4x47r: Warning Unhealthy (x7) 'Readiness probe failed: HTTP probe failed with statuscode: 500' at 17:32:25Z
- pod_logs external-secrets-cert-controller: healthz check failed at 17:32:00Z (x7) through 17:32:25Z, then resumed updating webhook configs at 17:35:37Z

## Cause

1. **The external-secrets-webhook Deployment (security/external-secrets-webhook) runs with replicas: 1 (single replica, no HA). During a period of widespread Karpenter spot instance terminations (at least 3 nodes terminated in the 'default' nodepool, 1 in 'agents-gvisor'), the node hosting the webhook pod ip-10-0-2-98 went NotReady at ~16:22. The webhook pod q68qn was stuck on the terminating node for ~70 minutes before being force-evicted at 17:31:59. With only 1 replica and no healthy endpoints, the webhook Service 'external-secrets-webhook' returned 'no endpoints available', causing the Kustomization/zitadel server-side dry-run to fail when validating ExternalSecret/security/zitadel-envvars. The PodDisruptionBudget 'external-secrets-pdb' (minAvailable: 1) only blocks voluntary disruptions (node drains) — spot terminations are involuntary and bypass PDBs entirely.** (82%) — change: No GitOps change; Karpenter spot instance termination at ~16:22 (BidEvictedEvent at 17:34:00, node ip-10-0-2-98 NotReady at 16:22)
2. **Compounding factor: the cert-controller pod (external-secrets-cert-controller-9667b79b6-4x47r) also experienced readiness probe failures (HTTP 500) at 17:32:00-17:32:25, likely because its healthz check depends on the webhook being reachable. This amplified the disruption window — both the validating webhook and the cert rotation controller were unhealthy simultaneously during the node churn.** (30%)

## Resolution

- Scale the external-secrets-webhook Deployment to at least 2 replicas with pod anti-affinity (topologyKey: kubernetes.io/hostname) so that a single node termination cannot take all webhook endpoints down. The PDB (external-secrets-pdb) currently has minAvailable:1 but covers the external-secrets controller, not specifically the webhook — and PDBs do not protect against involuntary spot terminations anyway. Additionally, consider scheduling the webhook pods on on-demand nodes (via nodeSelector/nodeAffinity) rather than spot instances, since it is a critical admission webhook that blocks all ExternalSecret operations when unavailable. The Helm chart for external-secrets supports webhook.replicas and webhook.affinity values. (reversible=true)
- Review the external-secrets-cert-controller's healthz/readiness probe configuration — if it depends on the webhook being reachable, it will fail in lockstep with the webhook during node churn. Consider making the cert-controller's readiness probe independent of webhook availability, or running the cert-controller with multiple replicas as well. (reversible=true)

## Unresolved

- The exact reason the webhook pod q68qn remained stuck on the terminating node for ~70 minutes (16:22 to 17:31:59) before being force-evicted is unclear. The node ip-10-0-2-98 went NotReady at 16:22, but the pod was only Evicted at 17:31:59. This could be due to the node being in a NotReady but not fully terminated state, kubelet grace period, or pod's terminationGracePeriodSeconds. This extended the disruption window significantly.

## Citations

[1] No GitOps change; Karpenter spot instance termination at ~16:22 (BidEvictedEvent at 17:34:00, node ip-10-0-2-98 NotReady at 16:22)

