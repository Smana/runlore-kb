---
type: Incident
title: Kustomization/flux-operator ReconciliationFailed — spot instance terminations took down external-secrets webhook
description: 'AWS Spot Instance termination wave (Server.SpotInstanceTermination) evicted Karpenter nodepool ''default'' spot nodes starting ~16:18, taking down the external-secrets-webhook pod (Deployment/security/external-secrets-webhook) and leaving Service/security/external-secrets-webhook with no endpoints. With the validating webhook unavailable, Flux kustomize-controller''s server-side dry-run of ExternalSecret/flux-system/flux-ui-oidc was rejected with ''failed calling webhook validate.externalsecret.external-secrets.io: no endpoints available for service external-secrets-webhook'', causing Kustomization/flux-operator ReconciliationFailed at 16:21:44. The incident self-healed once the webhook pod rescheduled to ip-10-0-14-63 and started at 16:21:38 — reconciliation succeeded at 16:22:44. Karpenter reports 3 spot instances terminated in the ''default'' nodepool, and 4 nodes (ip-10-0-43-91, ip-10-0-6-115, ip-10-0-46-64, ip-10-0-3-118) went NotReady simultaneously starting 16:18. This is a recurrence of the same class of problem documented in the past incident (node rotation evicting admission webhooks), though here the trigger is spot reclamation rather than EKS managed-nodegroup rotation.'
resource: security/external-secrets-webhook
alert_resource: flux-system/flux-operator
tags:
    - runlore
    - incident
    - pod
    - security
timestamp: "2026-11-17T16:35:36Z"
fingerprint: 08b56c25741abc16d2612ba27b0c9c1a53594e62761161daa48a330eb0b2ac3a
confidence: 0.82
provenance:
    - No GitOps change; AWS spot instance reclamation (Server.SpotInstanceTermination) on Karpenter default nodepool at ~16:18 UTC
origin: runlore
---

## Decision

- **why keep:** AWS Spot Instance termination wave (Server.SpotInstanceTermination) evicted Karpenter nodepool 'default' spot nodes starting ~16:18, taking down the external-secrets-webhook pod (Deployment/security/external-secrets-webhook) and leaving Service/security/external-secrets-webhook with no endpoints. With the validating webhook unavailable, Flux kustomize-controller's server-side dry-run of ExternalSecret/flux-system/flux-ui-oidc was rejected with 'failed calling webhook validate.externalsecret.external-secrets.io: no endpoints available for service external-secrets-webhook', causing Kustomization/flux-operator ReconciliationFailed at 16:21:44. The incident self-healed once the webhook pod rescheduled to ip-10-0-14-63 and started at 16:21:38 — reconciliation succeeded at 16:22:44. Karpenter reports 3 spot instances terminated in the 'default' nodepool, and 4 nodes (ip-10-0-43-91, ip-10-0-6-115, ip-10-0-46-64, ip-10-0-3-118) went NotReady simultaneously starting 16:18. This is a recurrence of the same class of problem documented in the past incident (node rotation evicting admission webhooks), though here the trigger is spot reclamation rather than EKS managed-nodegroup rotation.
- **confidence:** 82%
- **provenance:** No GitOps change; AWS spot instance reclamation (Server.SpotInstanceTermination) on Karpenter default nodepool at ~16:18 UTC

## Symptom

Kustomization/flux-operator ReconciliationFailed — spot instance terminations took down external-secrets webhook

Affected resource: Pod security/external-secrets-webhook-6b467fd6cd-q68qn

## Investigate

- cloud_resource_health: Karpenter nodepool 'default: instances=9 spot=9 terminated=3 spot-term-reasons=[Server.SpotInstanceTermination]' — 3 spot instances were reclaimed by AWS
- query_metrics_range node Ready: 4 nodes went NotReady simultaneously — ip-10-0-43-91 (NotReady max=1 @16:18:15, recovered 16:18:33), ip-10-0-6-115 (NotReady max=1 @16:19:54, recovered 16:20:15), ip-10-0-46-64 (NotReady max=1 @16:21:54, recovered 16:22:33), ip-10-0-3-118 (NotReady max=1 @16:22:33, recovered 16:22:54)
- kube_events (security ns): 'Pod/external-secrets-webhook-6b467fd6cd-5f6ff Unhealthy (x12): Readiness probe failed: dial tcp 100.64.33.174:8081: connect: connection refused' at 16:21:30 — the old webhook pod was failing health probes
- kube_events (flux-system ns): 'Kustomization/flux-operator ReconciliationFailed: ExternalSecret/flux-system/flux-ui-oidc dry-run failed (InternalError): no endpoints available for service external-secrets-webhook' at 16:21:44
- kube_events (flux-system ns): Multiple other Kustomizations failed simultaneously with the same pattern — observability-victoria-metrics-k8s-stack at 16:21:35 (external-secrets webhook), security-public-certs at 16:21:55 and 16:22:25 (cert-manager webhook) — indicating a cluster-wide webhook outage, not a flux-operator-specific issue
- kube_events (security ns): 'Pod/external-secrets-webhook-6b467fd6cd-q68qn Scheduled: Successfully assigned security/external-secrets-webhook-6b467fd6cd-q68qn to ip-10-0-14-63' at 16:21:29, 'Container started' at 16:21:38 — webhook pod rescheduled and recovered
- gitops_resource_status: Kustomization flux-system/flux-operator now Ready=True (ReconciliationSucceeded) — last ReconciliationFailed at 16:21:44, first post-incident ReconciliationSucceeded at 16:22:44 (492 prior successes combined in event), confirming self-healing
- pod_status (security ns): external-secrets-webhook-6b467fd6cd-q68qn Running ready=1/1 age=11m on ip-10-0-14-63 — pod is healthy now, confirming recovery
- kube_events (security ns): 'Pod/kyverno-reports-controller-5959766b88-96vsr FailedCreatePodSandBox: failed to setup network for sandbox: plugin type="cilium-cni" failed (add): unable to allocate IP via local cilium agent: [POST /ipam][502] postIpamFailure "all CIDR ranges are exhausted"' at 16:21:30 and 16:21:42
- kube_events (security ns): 'Pod/external-secrets-cert-controller-9667b79b6-xkmpr Unhealthy: Readiness probe failed: context deadline exceeded' at 16:21:36 — cert-controller also delayed in coming ready

## Cause

1. **AWS Spot Instance termination wave (Server.SpotInstanceTermination) evicted Karpenter nodepool 'default' spot nodes starting ~16:18, taking down the external-secrets-webhook pod (Deployment/security/external-secrets-webhook) and leaving Service/security/external-secrets-webhook with no endpoints. With the validating webhook unavailable, Flux kustomize-controller's server-side dry-run of ExternalSecret/flux-system/flux-ui-oidc was rejected with 'failed calling webhook validate.externalsecret.external-secrets.io: no endpoints available for service external-secrets-webhook', causing Kustomization/flux-operator ReconciliationFailed at 16:21:44. The incident self-healed once the webhook pod rescheduled to ip-10-0-14-63 and started at 16:21:38 — reconciliation succeeded at 16:22:44. Karpenter reports 3 spot instances terminated in the 'default' nodepool, and 4 nodes (ip-10-0-43-91, ip-10-0-6-115, ip-10-0-46-64, ip-10-0-3-118) went NotReady simultaneously starting 16:18. This is a recurrence of the same class of problem documented in the past incident (node rotation evicting admission webhooks), though here the trigger is spot reclamation rather than EKS managed-nodegroup rotation.** (82%) — change: No GitOps change; AWS spot instance reclamation (Server.SpotInstanceTermination) on Karpenter default nodepool at ~16:18 UTC
2. **Cilium IPAM exhaustion ('all CIDR ranges are exhausted') on recovering nodes delayed pod sandbox creation for security-namespace pods (kyverno-reports-controller, external-secrets-cert-controller), extending the window during which admission webhooks were unavailable. This is a compounding factor rather than the root trigger — the spot terminations initiated the disruption, but IP exhaustion slowed recovery by preventing new pods from getting network IPs.** (30%)

## Resolution

- No action suggested by the investigation — see `## Unresolved`.

## Unresolved

- Whether the spot termination wave was a routine AWS capacity reclamation or triggered by a broader AZ capacity event could not be determined from available tools.

## Citations

[1] No GitOps change; AWS spot instance reclamation (Server.SpotInstanceTermination) on Karpenter default nodepool at ~16:18 UTC

