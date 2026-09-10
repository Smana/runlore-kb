---
type: Incident
title: flux-notifications ReconciliationFailed — external-secrets-webhook had no endpoints during Cilium IPAM exhaustion…
description: The external-secrets-webhook pod restart hit a Cilium IPAM exhaustion window on node ip-10-0-16-32, leaving the external-secrets-webhook Service with no endpoints for ~90s. During that window, Flux's dry-run apply of ExternalSecret/flux-system/flux-slack-app failed because the validating webhook (validate.externalsecret.external-secrets.io) could not be reached — the error 'no endpoints available for service external-secrets-webhook'. The webhook pod eventually obtained an IP at 16:26:09, started serving, and the next Flux reconciliation at 16:50:11 succeeded, so this is a transient failure that has already self-healed.
resource: flux-system/flux-notifications
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-09-10T16:59:17Z"
fingerprint: 78726a78e22d98a584a4231aa745ab35d1fa93cac05ed62f779b021e837215a7
confidence: 0.85
---

## Decision

- **why keep:** The external-secrets-webhook pod restart hit a Cilium IPAM exhaustion window on node ip-10-0-16-32, leaving the external-secrets-webhook Service with no endpoints for ~90s. During that window, Flux's dry-run apply of ExternalSecret/flux-system/flux-slack-app failed because the validating webhook (validate.externalsecret.external-secrets.io) could not be reached — the error 'no endpoints available for service external-secrets-webhook'. The webhook pod eventually obtained an IP at 16:26:09, started serving, and the next Flux reconciliation at 16:50:11 succeeded, so this is a transient failure that has already self-healed.
- **confidence:** 85%

## Symptom

flux-notifications ReconciliationFailed — external-secrets-webhook had no endpoints during Cilium IPAM exhaustion window

Affected resource: Kustomization flux-system/flux-notifications

## Investigate

- kube_events (security, object=external-secrets-webhook-848cbfdb57-v9xff): 16:24:38 FailedCreatePodSandBox 'cilium-cni failed (add): unable to allocate IP via local cilium agent: [POST /ipam][502] postIpamFailure "all CIDR ranges are exhausted"', repeated 16:24:53
- kube_events: 16:24:38 'Pod/external-secrets-webhook-848cbfdb57-wdnfs Killing: Stopping container webhook' and 'ReplicaSet/external-secrets-webhook-848cbfdb57 SuccessfulCreate: Created pod: external-secrets-webhook-848cbfdb57-v9xff' — the old pod was killed and a new one created that could not get an IP
- kube_events: 16:26:05 'Pulling image' → 16:26:08 'Created/Pulled' → 16:26:09 'Started: Container started' — pod eventually got an IP and started ~90s after the first failure
- pod_logs (external-secrets-webhook): 16:26:10 'Starting webhook server' / 'Registering webhook path /validate-external-secrets-io-v1-externalsecret' — webhook began serving, endpoints came back
- kube_events (flux-system, object=flux-notifications): 16:25:30 'ReconciliationFailed: ExternalSecret/flux-system/flux-slack-app dry-run failed (InternalError): ... failed calling webhook validate.externalsecret.external-secrets.io ... no endpoints available for service external-secrets-webhook' — the alert timestamp falls within the no-endpoints window
- gitops_resource_status (Kustomization flux-system/flux-notifications): Ready=True (ReconciliationSucceeded) at 16:50:11 — the failure cleared on its own once the webhook pod recovered
- query_metrics_range (cilium_operator_ipam_nodes by category): at-capacity=0, in-deficit=0, total=6 now — no nodes currently at capacity, the exhaustion was transient
- pod_status (security): external-secrets-webhook-848cbfdb57-v9xff Running ready=1/1 age=27m — webhook pod is healthy now
- pod_status (security): exactly one external-secrets-webhook pod (external-secrets-webhook-848cbfdb57-v9xff) — single replica, no HA
- resource_spec (Service external-secrets-webhook): selector app.kubernetes.io/name=external-secrets-webhook — the service endpoints are backed only by this single-replica deployment
- The 16:24:37-16:26:09 window where the old webhook pod was killed and the new one had no IP is exactly when the Service had zero endpoints and the ExternalSecret dry-run failed

## Cause

1. **The external-secrets-webhook pod restart hit a Cilium IPAM exhaustion window on node ip-10-0-16-32, leaving the external-secrets-webhook Service with no endpoints for ~90s. During that window, Flux's dry-run apply of ExternalSecret/flux-system/flux-slack-app failed because the validating webhook (validate.externalsecret.external-secrets.io) could not be reached — the error 'no endpoints available for service external-secrets-webhook'. The webhook pod eventually obtained an IP at 16:26:09, started serving, and the next Flux reconciliation at 16:50:11 succeeded, so this is a transient failure that has already self-healed.** (85%)
2. **The external-secrets-webhook Deployment has a single replica, so any restart of that one pod creates a no-endpoints window for the validating webhook service. This made the Cilium IPAM exhaustion window (which would otherwise be a brief pod-startup delay) sufficient to break ExternalSecret validation cluster-wide and fail Flux reconciliations that dry-run apply ExternalSecrets.** (55%)

## Resolution

- No immediate action required — the failure self-healed and all pods are healthy. Monitor cilium_operator_ipam_nodes{category="at-capacity"} > 0 to catch recurrence before it blocks another webhook-bearing pod. If it recurs, free an IP on the affected node (delete a replicated pod that reschedules elsewhere) or move control-plane-ish pods off ENI-IPAM-limited instance types. (reversible=true)
- Increase the external-secrets-webhook Deployment replica count to ≥2 with anti-affinity so a single pod restart cannot empty the Service's endpoints, making ExternalSecret validation resilient to transient pod/CNI failures. (reversible=true)

## Unresolved

- What triggered the external-secrets-webhook and cert-controller pod restarts at 16:24:37 — was it a spot instance interruption, a node drain, or a Karpenter consolidation event? The pod events show 'Killing' but the cloud-side cause could not be retrieved due to CloudTrail lookup timeouts.

