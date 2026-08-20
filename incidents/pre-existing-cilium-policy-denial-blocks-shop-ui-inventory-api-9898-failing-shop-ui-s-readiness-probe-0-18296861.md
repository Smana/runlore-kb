---
type: Incident
title: '[PRE-EXISTING] Cilium policy denial blocks shop-ui→inventory-api:9898, failing shop-ui''s readiness probe (0…'
description: Cilium network policy denial is still blocking traffic from shop-ui to inventory-api on port 9898. The shop-ui pod is Running (process up, 0 restarts) but its readiness probe — which does `curl -sf --max-time 2 http://inventory-api.demo.svc.cluster.local:9898/readyz` — fails because the connection is denied by Cilium policy (POLICY_DENIED). This leaves the Deployment with 0 available replicas. The inventory-api pod is healthy (Running ready=1/1). This is the SAME fault identified 33 minutes ago and it has NOT been fixed — network drops are still actively occurring (76 POLICY_DENIED events, first 10:43:34Z → last 10:47:13Z).
resource: demo/shop-ui
tags:
    - runlore
    - incident
    - deployment
    - demo
timestamp: "2026-08-20T10:48:55Z"
fingerprint: 18296861205a5cf4e6cb52a6da1c67652e4007ae9091137be15295d3ff38cd2f
confidence: 0.9
provenance:
    - No GitOps or cloud change detected — the Cilium policy is either pre-existing or was applied out-of-band (the demo namespace is not managed by this GitOps engine; what_changed returned no managed objects for demo)
---

## Decision

- **why keep:** Cilium network policy denial is still blocking traffic from shop-ui to inventory-api on port 9898. The shop-ui pod is Running (process up, 0 restarts) but its readiness probe — which does `curl -sf --max-time 2 http://inventory-api.demo.svc.cluster.local:9898/readyz` — fails because the connection is denied by Cilium policy (POLICY_DENIED). This leaves the Deployment with 0 available replicas. The inventory-api pod is healthy (Running ready=1/1). This is the SAME fault identified 33 minutes ago and it has NOT been fixed — network drops are still actively occurring (76 POLICY_DENIED events, first 10:43:34Z → last 10:47:13Z).
- **confidence:** 90%
- **provenance:** No GitOps or cloud change detected — the Cilium policy is either pre-existing or was applied out-of-band (the demo namespace is not managed by this GitOps engine; what_changed returned no managed objects for demo)

## Symptom

[PRE-EXISTING] Cilium policy denial blocks shop-ui→inventory-api:9898, failing shop-ui's readiness probe (0 available replicas) — still unfixed

Affected resource: Deployment demo/shop-ui

## Investigate

- network_drops (demo/shop-ui-585d9d8ddc-8858j): 76 DROPPED (POLICY_DENIED) from demo/shop-ui → demo/inventory-api, first 2026-08-20T10:43:34Z → last 2026-08-20T10:47:13Z — still ongoing at investigation time
- pod_status: shop-ui-585d9d8ddc-8858j Running ready=0/1 age=43m (no restarts, process is up); inventory-api-c4c5bf4dd-7mnnx Running ready=1/1 (healthy)
- resource_spec Deployment demo/shop-ui: readiness probe is `exec: curl -sf --max-time 2 http://inventory-api.demo.svc.cluster.local:9898/readyz` — probe calls inventory-api:9898, exactly the destination being denied
- kube_events: Pod/shop-ui-585d9d8ddc-8858j Unhealthy x255: 'Readiness probe failed' (10:43:36Z onward)
- alert_rule ShopUiUnavailable: expr `kube_deployment_status_replicas_available{namespace="demo", deployment="shop-ui"} == 0` for=2m, firing
- query_metrics: kube_deployment_status_replicas_available{namespace="demo", deployment="shop-ui"} = 0 (confirmed); range query shows value=0 for entire 60-min window
- Deployment status: Available=False (MinimumReplicasUnavailable), Progressing=False (ProgressDeadlineExceeded), unavailableReplicas=1

## Cause

1. **Cilium network policy denial is still blocking traffic from shop-ui to inventory-api on port 9898. The shop-ui pod is Running (process up, 0 restarts) but its readiness probe — which does `curl -sf --max-time 2 http://inventory-api.demo.svc.cluster.local:9898/readyz` — fails because the connection is denied by Cilium policy (POLICY_DENIED). This leaves the Deployment with 0 available replicas. The inventory-api pod is healthy (Running ready=1/1). This is the SAME fault identified 33 minutes ago and it has NOT been fixed — network drops are still actively occurring (76 POLICY_DENIED events, first 10:43:34Z → last 10:47:13Z).** (90%) — change: No GitOps or cloud change detected — the Cilium policy is either pre-existing or was applied out-of-band (the demo namespace is not managed by this GitOps engine; what_changed returned no managed objects for demo)

## Resolution

- Fix or create a CiliumNetworkPolicy (or CiliumClusterwideNetworkPolicy) that allows egress from shop-ui (or any pod with label app=shop-ui) to inventory-api on port 9898 in the demo namespace. The policy currently enforcing the denial could not be found by common names (inventory-api, default-deny, allow-egress, shop-ui, demo) — it may have an unusual name, be cluster-scoped, or be a stale Cilium enforcement rule. The operator should run `kubectl get cnp,ccnp -A` to list all Cilium policies and identify which one denies shop-ui→inventory-api:9898, then either add an allow rule for that flow or remove/patch the deny policy. Once traffic flows, the readiness probe will succeed and the Deployment will become available automatically. (reversible=true)

## Unresolved

- Why was the Cilium policy denying shop-ui→inventory-api:9898 created or left in place? The demo namespace is not managed by this GitOps engine, so the policy was applied out-of-band. The intent (accidental vs. intentional, leftover from testing, etc.) can only be determined by the human who applied it.

## Citations

[1] No GitOps or cloud change detected — the Cilium policy is either pre-existing or was applied out-of-band (the demo namespace is not managed by this GitOps engine; what_changed returned no managed objects for demo)

