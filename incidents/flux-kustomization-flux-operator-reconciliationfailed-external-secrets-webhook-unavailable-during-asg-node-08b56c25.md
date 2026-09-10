---
type: Incident
title: Flux Kustomization flux-operator ReconciliationFailed — external-secrets webhook unavailable during ASG node…
description: ASG node rotation terminated a node hosting the external-secrets-webhook pod. The pod was rescheduled onto a newly-joined node (ip-10-0-16-32) whose Cilium IPAM pool had not yet been allocated (cilium_ipam_capacity=0). CNI ADD returned 502 'all CIDR ranges are exhausted' for ~90 seconds, during which the validating webhook had no endpoints. This caused ExternalSecret dry-run validation to fail, cascading to Kustomization reconciliation failures for flux-operator, zitadel, flux-notifications, observability, and observability-victoria-metrics-k8s-stack. The system self-healed once Cilium allocated the IPAM pool (capacity 0→11 at 16:26Z).
resource: security/external-secrets-webhook
alert_resource: flux-system/flux-operator
tags:
    - runlore
    - incident
    - deployment
    - security
timestamp: "2026-09-10T16:41:56Z"
fingerprint: 08b56c25741abc16d2612ba27b0c9c1a53594e62761161daa48a330eb0b2ac3a
confidence: 0.76
provenance:
    - ASG node rotation (CompleteLifecycleAction at 16:26:39Z, TerminateInstances at 16:26:40Z) — no GitOps change involved
---

## Decision

- **why keep:** ASG node rotation terminated a node hosting the external-secrets-webhook pod. The pod was rescheduled onto a newly-joined node (ip-10-0-16-32) whose Cilium IPAM pool had not yet been allocated (cilium_ipam_capacity=0). CNI ADD returned 502 'all CIDR ranges are exhausted' for ~90 seconds, during which the validating webhook had no endpoints. This caused ExternalSecret dry-run validation to fail, cascading to Kustomization reconciliation failures for flux-operator, zitadel, flux-notifications, observability, and observability-victoria-metrics-k8s-stack. The system self-healed once Cilium allocated the IPAM pool (capacity 0→11 at 16:26Z).
- **confidence:** 76%
- **provenance:** ASG node rotation (CompleteLifecycleAction at 16:26:39Z, TerminateInstances at 16:26:40Z) — no GitOps change involved

## Symptom

Flux Kustomization flux-operator ReconciliationFailed — external-secrets webhook unavailable during ASG node rotation (Cilium IPAM not ready on new node)

Affected resource: Deployment security/external-secrets-webhook

## Investigate

- kube_events (security): Pod/external-secrets-webhook-848cbfdb57-v9xff FailedCreatePodSandBox at 16:24:38Z and 16:24:53Z: 'failed to setup network for sandbox ... plugin type="cilium-cni" failed (add): unable to allocate IP via local cilium agent: [POST /ipam][502] postIpamFailure "all CIDR ranges are exhausted"'
- kube_events (security): Pod scheduled at 16:24:38Z to ip-10-0-16-32.eu-west-3.compute.internal, then container Created/Started at 16:26:05-09Z — confirming ~90s gap with no running webhook pod
- query_metrics_range cilium_ipam_capacity: node ip-10-0-16-32 had first=0, biggest jump +11 at 16:26Z — the new node's IPAM pool was empty until Cilium operator allocated it
- query_metrics_range cilium_ip_addresses: node ip-10-0-25-185 dropped from 26 to 7 allocated IPs at 16:25Z (biggest jump -19) — the old node was being drained/terminated, forcing pod rescheduling
- incident_timeline: [cloud] ASG CompleteLifecycleAction at 16:26:39Z and EC2 TerminateInstances by AutoScaling at 16:26:40Z — ASG node rotation triggered the rescheduling
- incident_timeline: [event] ReconciliationFailed for flux-operator at 16:25:18Z, zitadel at 16:25:23Z, flux-notifications at 16:25:30Z, observability at 16:25:35Z, observability-victoria-metrics-k8s-stack at 16:25:39Z — all citing the same 'no endpoints available for service external-secrets-webhook' error
- gitops_resource_status Kustomization flux-operator: Ready=True with recovery events — '16:25:18Z Warning ReconciliationFailed ... no endpoints available' then '16:29:52Z Normal ReconciliationSucceeded (x44)' — the Kustomization self-healed
- pod_status (security): external-secrets-webhook-848cbfdb57-v9xff Running ready=1/1 age=9m (at time of check) — confirms the pod recovered
- resource_spec Deployment security/external-secrets-webhook: spec.replicas: 1 — single replica with no anti-affinity or topology spread constraints
- pod_status (security): only one external-secrets-webhook pod exists (external-secrets-webhook-848cbfdb57-v9xff) — no redundancy
- incident_timeline: five separate Kustomizations (flux-operator, zitadel, flux-notifications, observability, observability-victoria-metrics-k8s-stack) all failed within 21 seconds of each other (16:25:18–16:25:39) — a single webhook pod going down cascaded across the entire cluster's GitOps reconciliation

## Cause

1. **ASG node rotation terminated a node hosting the external-secrets-webhook pod. The pod was rescheduled onto a newly-joined node (ip-10-0-16-32) whose Cilium IPAM pool had not yet been allocated (cilium_ipam_capacity=0). CNI ADD returned 502 'all CIDR ranges are exhausted' for ~90 seconds, during which the validating webhook had no endpoints. This caused ExternalSecret dry-run validation to fail, cascading to Kustomization reconciliation failures for flux-operator, zitadel, flux-notifications, observability, and observability-victoria-metrics-k8s-stack. The system self-healed once Cilium allocated the IPAM pool (capacity 0→11 at 16:26Z).** (76%) — change: ASG node rotation (CompleteLifecycleAction at 16:26:39Z, TerminateInstances at 16:26:40Z) — no GitOps change involved
2. **The external-secrets-webhook Deployment runs as a single replica (spec.replicas: 1), making the admission webhook a single point of failure. Any pod restart, node rotation, or brief scheduling delay creates a window where the validating webhook has zero endpoints, blocking all ExternalSecret operations and cascading to Flux Kustomization failures across multiple namespaces.** (50%)

## Resolution

- No immediate action needed — the system self-healed. To prevent recurrence: (1) scale external-secrets-webhook to replicas: 2+ with podAntiAffinity/topologySpreadConstraints so a single node rotation cannot take the admission webhook offline; (2) consider setting failurePolicy: Ignore on the validate.externalsecret webhook (or adding a timeout-based fallback) so transient webhook unavailability doesn't block all Kustomization reconciliations; (3) monitor cilium_operator_ipam_nodes{category="at-capacity"} and new-node IPAM readiness to catch IPAM allocation delays before they impact critical webhooks. (reversible=true)
- Increase external-secrets-webhook replicas to at least 2 and add podAntiAffinity (preferredDuringScheduling) or topologySpreadConstraints to ensure replicas land on different nodes. This is a standard HA pattern for admission webhooks and would have entirely prevented this incident. (reversible=false)

## Unresolved

- Whether the ASG node rotation was a scheduled/karpenter-driven event or a manual action — the CloudTrail shows CompleteLifecycleAction and TerminateInstances by AutoScaling, suggesting an ASG-driven rotation, but the originating trigger (spot interruption, health check failure, scheduled replacement) was not visible in the available tools

## Citations

[1] ASG node rotation (CompleteLifecycleAction at 16:26:39Z, TerminateInstances at 16:26:40Z) — no GitOps change involved

