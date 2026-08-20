---
type: Incident
title: '[PRE-EXISTING #2] CheckoutUiUnavailable: CiliumNetworkPolicy still denies checkout-ui→orders-api traffic, failing…'
description: The CiliumNetworkPolicy demo/orders-api-allow-payments-only is STILL active and STILL denies checkout-ui's traffic to orders-api, exactly as concluded in the previous investigation ~1h13m ago — the fault was never remediated. The policy selects orders-api endpoints (endpointSelector matchLabels app=orders-api) and permits ingress ONLY from app=payments-worker pods on TCP/9898. checkout-ui (app=checkout-ui) is not in the allowlist, so Cilium drops its readiness-probe calls to orders-api (POLICY_DENIED), the probe never succeeds, the pod stays Running/ready=0/1, and the Deployment reports 0 available replicas. The policy's Valid condition lastTransitionTime is 2026-08-20T11:09:54Z — it has been enforced continuously since well before the first occurrence.
resource: demo/orders-api-allow-payments-only
alert_resource: demo/checkout-ui
tags:
    - runlore
    - incident
    - ciliumnetworkpolicy
    - demo
timestamp: "2026-08-20T12:31:24Z"
fingerprint: 81fe63375a97dfe569f113771435ae6c32909e2066e26e201526db02d0089cd4
confidence: 0.8
provenance:
    - CiliumNetworkPolicy demo/orders-api-allow-payments-only validated at 2026-08-20T11:09:54Z (pre-existing since first occurrence)
---

## Decision

- **why keep:** The CiliumNetworkPolicy demo/orders-api-allow-payments-only is STILL active and STILL denies checkout-ui's traffic to orders-api, exactly as concluded in the previous investigation ~1h13m ago — the fault was never remediated. The policy selects orders-api endpoints (endpointSelector matchLabels app=orders-api) and permits ingress ONLY from app=payments-worker pods on TCP/9898. checkout-ui (app=checkout-ui) is not in the allowlist, so Cilium drops its readiness-probe calls to orders-api (POLICY_DENIED), the probe never succeeds, the pod stays Running/ready=0/1, and the Deployment reports 0 available replicas. The policy's Valid condition lastTransitionTime is 2026-08-20T11:09:54Z — it has been enforced continuously since well before the first occurrence.
- **confidence:** 80%
- **provenance:** CiliumNetworkPolicy demo/orders-api-allow-payments-only validated at 2026-08-20T11:09:54Z (pre-existing since first occurrence)

## Symptom

[PRE-EXISTING #2] CheckoutUiUnavailable: CiliumNetworkPolicy still denies checkout-ui→orders-api traffic, failing readiness probe

Affected resource: CiliumNetworkPolicy demo/orders-api-allow-payments-only

## Investigate

- Alert rule CheckoutUiUnavailable: kube_deployment_status_replicas_available{namespace="demo", deployment="checkout-ui"} == 0, for=2m — fires on 0 available replicas
- pod_status: checkout-ui-8477c74c86-pxf4k Running ready=0/1 (not Ready); orders-api-648577f765-h667z Running ready=1/1 (healthy — downstream is not the problem)
- kube_events: Pod/checkout-ui-8477c74c86-pxf4k Unhealthy (x23): 'Readiness probe failed:' starting 12:28:22Z; prior pod checkout-ui-8477c74c86-2vzct had the same (x452) from 12:20:22Z
- network_drops: demo/checkout-ui-8477c74c86-pxf4k -> demo/orders-api-648577f765-h667z DROPPED (POLICY_DENIED) x94, first 12:24:59Z → last 12:28:30Z — Cilium policy enforcement is denying the flow
- CiliumNetworkPolicy demo/orders-api-allow-payments-only spec: endpointSelector app=orders-api; ingress fromEndpoints app=payments-worker only, toPorts TCP/9898 — checkout-ui (app=checkout-ui) is NOT in the allowlist
- Policy status condition Valid=True, lastTransitionTime 2026-08-20T11:09:54Z — policy has been enforced continuously since before the first occurrence, confirming it was never fixed

## Cause

1. **The CiliumNetworkPolicy demo/orders-api-allow-payments-only is STILL active and STILL denies checkout-ui's traffic to orders-api, exactly as concluded in the previous investigation ~1h13m ago — the fault was never remediated. The policy selects orders-api endpoints (endpointSelector matchLabels app=orders-api) and permits ingress ONLY from app=payments-worker pods on TCP/9898. checkout-ui (app=checkout-ui) is not in the allowlist, so Cilium drops its readiness-probe calls to orders-api (POLICY_DENIED), the probe never succeeds, the pod stays Running/ready=0/1, and the Deployment reports 0 available replicas. The policy's Valid condition lastTransitionTime is 2026-08-20T11:09:54Z — it has been enforced continuously since well before the first occurrence.** (80%) — change: CiliumNetworkPolicy demo/orders-api-allow-payments-only validated at 2026-08-20T11:09:54Z (pre-existing since first occurrence)

## Resolution

- Add an ingress rule to the CiliumNetworkPolicy demo/orders-api-allow-payments-only that permits traffic from app=checkout-ui pods (fromEndpoints matchLabels app=checkout-ui) to TCP/9898, matching the existing payments-worker rule. Alternatively, if checkout-ui should reach all in-namespace services, broaden the fromEndpoints. After the policy update, the readiness probe should succeed within its next check window and the Deployment will report available replicas. (reversible=true)

## Citations

[1] CiliumNetworkPolicy demo/orders-api-allow-payments-only validated at 2026-08-20T11:09:54Z (pre-existing since first occurrence)

