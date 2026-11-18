---
type: Incident
title: Kustomization/envoy-gateway ReconciliationFailed — external-secrets-webhook pod evicted during Karpenter spot node…
description: 'Karpenter spot node ip-10-0-9-1.eu-west-3.compute.internal was terminated (BidEvictedEvent at 20:52:26Z, Karpenter nodepool default shows terminated=1), causing the kubelet to forcefully evict the external-secrets-webhook pod external-secrets-webhook-6b467fd6cd-d5lv5 at 20:50:25Z (''Evicted pod: Forceful Termination'' + ''Killing: Stopping container webhook''). The replacement pod 6tkj7 was scheduled at 20:50:25Z on new node ip-10-0-41-105 but did not become ready until 20:51:46Z (webhook server started at 20:51:07Z). During the ~42-second gap with zero ready webhook pods, Flux''s Kustomization envoy-gateway reconciliation at 20:50:36Z attempted to dry-run apply ExternalSecret/envoy-gateway-system/ai-gateway-ratelimit-valkey; the validating admission webhook call to external-secrets-webhook.security.svc:443 failed with ''no endpoints available for service external-secrets-webhook''. The webhook has since recovered and the ExternalSecret itself is healthy (status Ready=True, SecretSynced).'
resource: security/external-secrets-webhook
alert_resource: flux-system/envoy-gateway
tags:
    - runlore
    - incident
    - pod
    - security
timestamp: "2026-11-18T20:56:12Z"
fingerprint: f609bf2a4a60f0d413bff89f17984e514c58b641353b59a4750454ffd993bb44
confidence: 0.85
origin: runlore
---

## Decision

- **why keep:** Karpenter spot node ip-10-0-9-1.eu-west-3.compute.internal was terminated (BidEvictedEvent at 20:52:26Z, Karpenter nodepool default shows terminated=1), causing the kubelet to forcefully evict the external-secrets-webhook pod external-secrets-webhook-6b467fd6cd-d5lv5 at 20:50:25Z ('Evicted pod: Forceful Termination' + 'Killing: Stopping container webhook'). The replacement pod 6tkj7 was scheduled at 20:50:25Z on new node ip-10-0-41-105 but did not become ready until 20:51:46Z (webhook server started at 20:51:07Z). During the ~42-second gap with zero ready webhook pods, Flux's Kustomization envoy-gateway reconciliation at 20:50:36Z attempted to dry-run apply ExternalSecret/envoy-gateway-system/ai-gateway-ratelimit-valkey; the validating admission webhook call to external-secrets-webhook.security.svc:443 failed with 'no endpoints available for service external-secrets-webhook'. The webhook has since recovered and the ExternalSecret itself is healthy (status Ready=True, SecretSynced).
- **confidence:** 85%

## Symptom

Kustomization/envoy-gateway ReconciliationFailed — external-secrets-webhook pod evicted during Karpenter spot node termination (transient, ~42s webhook outage)

Affected resource: Pod security/external-secrets-webhook-6b467fd6cd-d5lv5

## Investigate

- kube_events (security, pod d5lv5): '2026-11-18T20:50:25Z Normal Evicted: Evicted pod: Forceful Termination' and 'Killing: Stopping container webhook' — the old webhook pod was force-killed by the node
- kube_events (security, pod 6tkj7): '2026-11-18T20:50:25Z Scheduled: Successfully assigned security/external-secrets-webhook-6b467fd6cd-6tkj7 to ip-10-0-41-105' and '2026-11-18T20:51:06Z Started: Container started' — replacement pod took 41s to start
- query_metrics_range kube_pod_info: old pod d5lv5 was on node='ip-10-0-9-1.eu-west-3.compute.internal' (host_ip=10.0.9.1) — the node that went NotReady
- query_metrics_range node Ready condition: ip-10-0-9-1 NotReady max=1@2026-11-18T20:53:00Z — the node hosting the webhook pod went down
- cloud_what_changed: 'BidEvictedEvent' at 20:52:26Z and Karpenter CreateFleet/CreateLaunchTemplate/RunInstances calls at 20:51:39-20:52:40 — active node replacement by Karpenter
- cloud_resource_health: 'Karpenter nodepool default: instances=7 spot=7 terminated=1' — a spot node was terminated
- gitops_resource_status Kustomization/envoy-gateway: Ready=False (ReconciliationFailed), message 'ExternalSecret/envoy-gateway-system/ai-gateway-ratelimit-valkey dry-run failed (InternalError): failed calling webhook validate.externalsecret.external-secrets.io: no endpoints available for service external-secrets-webhook' at 20:50:36Z
- kube_events (flux-system, envoy-gateway): '2026-11-18T20:45:46Z Normal ReconciliationSucceeded (x16)' then '2026-11-18T20:50:36Z Warning ReconciliationFailed' — the failure occurred in the reconciliation right after the eviction
- pod_logs (external-secrets-webhook-6tkj7): webhook server started at 20:51:07Z ('Starting webhook server', 'Serving webhook server') — confirming the ~42s gap with no webhook
- query_metrics_range kube_pod_container_status_ready webhook 6tkj7: first=0 at 20:50:46Z, became 1 at 20:51:46Z — 42 seconds with no ready webhook pod
- resource_spec ExternalSecret/envoy-gateway-system/ai-gateway-ratelimit-valkey: status.conditions Ready=True (SecretSynced), refreshTime 2026-11-17T08:01:57Z — the ExternalSecret itself is healthy, confirming the failure was the webhook, not the resource
- pod_status (security): only one external-secrets-webhook pod (6b467fd6cd-6tkj7) Running — single replica, no redundancy
- pod_status (security): external-secrets-webhook pod is on ip-10-0-41-105, a Karpenter spot node (cloud_resource_health: Karpenter nodepool default spot=7)
- past incident (kb_search): 'kyverno-admission-controller pods were simultaneously evicted by the same node rotation, causing no endpoints available for service kyverno-svc' — same pattern of webhook pods on rotating spot nodes without PDBs
- kube_events (security): '2026-11-18T20:50:26Z Pod/kyverno-admission-controller-5cccc5cd44-gkz6c FailedCreatePodSandBox: all CIDR ranges are exhausted' — the kyverno webhook was also being rescheduled during the same node churn event, confirming cluster-wide webhook disruption during node rotation

## Cause

1. **Karpenter spot node ip-10-0-9-1.eu-west-3.compute.internal was terminated (BidEvictedEvent at 20:52:26Z, Karpenter nodepool default shows terminated=1), causing the kubelet to forcefully evict the external-secrets-webhook pod external-secrets-webhook-6b467fd6cd-d5lv5 at 20:50:25Z ('Evicted pod: Forceful Termination' + 'Killing: Stopping container webhook'). The replacement pod 6tkj7 was scheduled at 20:50:25Z on new node ip-10-0-41-105 but did not become ready until 20:51:46Z (webhook server started at 20:51:07Z). During the ~42-second gap with zero ready webhook pods, Flux's Kustomization envoy-gateway reconciliation at 20:50:36Z attempted to dry-run apply ExternalSecret/envoy-gateway-system/ai-gateway-ratelimit-valkey; the validating admission webhook call to external-secrets-webhook.security.svc:443 failed with 'no endpoints available for service external-secrets-webhook'. The webhook has since recovered and the ExternalSecret itself is healthy (status Ready=True, SecretSynced).** (85%)
2. **The external-secrets-webhook Deployment (external-secrets-webhook-6b467fd6cd) runs with a single replica on spot nodes provisioned by Karpenter. With only one replica and no PodDisruptionBudget, any node termination immediately removes all webhook endpoints. This is the same architectural pattern that caused the past incident where Kyverno's admission-controller pods were evicted during node rotation, producing 'no endpoints available for service kyverno-svc' and blocking Flux reconciliations (documented in the pre-existing karpenter-nodepools ArtifactFailed incident). The cluster's critical admission webhooks (external-secrets, kyverno) lack PDBs and anti-affinity/replica spread that would maintain quorum during node churn.** (45%)

## Resolution

- The webhook has already self-healed. Trigger a manual Flux reconciliation to clear the stale Ready=False status: `flux -n flux-system reconcile kustomization envoy-gateway`. To prevent recurrence, add a PodDisruptionBudget (minAvailable: 1) for the external-secrets-webhook Deployment in the security namespace so at least one webhook pod stays ready during node drains, and/or schedule webhook pods on on-demand nodes or across multiple Karpenter nodepools to avoid single-spot-node concentration. Consider setting failurePolicy: IfNotPresent on the validate.externalsecret.external-secrets.io ValidatingWebhookConfiguration so that admission requests are not blocked during transient webhook outages. (reversible=true)
- Increase external-secrets-webhook Deployment replicas to at least 2 with topologySpreadConstraints or podAntiAffinity to ensure pods land on different nodes. Add a PodDisruptionBudget (minAvailable: 1) for the webhook Deployment. Apply the same pattern to kyverno-admission-controller and cert-manager-webhook. Consider scheduling these critical admission webhook pods on on-demand nodes or a dedicated nodepool exempt from spot interruption/rotation. (reversible=true)

## Unresolved

- The exact trigger for the Karpenter spot node termination of ip-10-0-9-1 could not be determined — the BidEvictedEvent at 20:52:26Z indicates a spot price-based eviction, but CloudTrail events were truncated at 25 results and the specific TerminateInstance or SpotInterruption event for this node may have been outside the returned window. The node went NotReady at 20:53:00Z, slightly after the pod eviction at 20:50:25Z, suggesting the node was already being drained/cordoned before the formal NotReady transition.
- Whether the Kustomization envoy-gateway has already self-healed on its next 5-minute reconciliation cycle — at the time of investigation the status still showed Ready=False from the 20:50:36Z failure, but the next scheduled reconciliation (~20:55:46Z based on the 5m interval from 20:45:46Z) should succeed now that the webhook is back up.

