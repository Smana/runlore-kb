---
type: Incident
title: vllm-semantic-router absent — new llm-platform umbrella Kustomization shipped suspended, blocking all LLM child…
description: 'A new Flux ''llm-platform'' umbrella Kustomization was committed to the cluster''s flux-system sync with spec.suspend: true and prune: true. This is an intentional opt-in gate: while suspended, Flux does NOT create any of the child Kustomizations under clusters/mycluster-0-llm-platform/ — explicitly including vllm-semantic-router. With prune: true, Flux actively removes any previously-applied children. The result: the vllm-semantic-router Kustomization, its namespace pods, and all LLM-platform resources are gone from the cluster, so vmagent has no up{} target to scrape → the critical ''no live up samples'' alert fires.'
resource: flux-system/vllm-semantic-router
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-07-07T09:59:30Z"
fingerprint: 180bbc9014faa5ed7cefa7dba88d7f43f7172adda89a6e996e3f74a48ff1b597
confidence: 0.85
provenance:
    - 'flux-system Kustomization sync introduced clusters/mycluster-0/llm-platform.yaml (new file) with spec.suspend: true, prune: true'
---

## Decision

- **why keep:** A new Flux 'llm-platform' umbrella Kustomization was committed to the cluster's flux-system sync with spec.suspend: true and prune: true. This is an intentional opt-in gate: while suspended, Flux does NOT create any of the child Kustomizations under clusters/mycluster-0-llm-platform/ — explicitly including vllm-semantic-router. With prune: true, Flux actively removes any previously-applied children. The result: the vllm-semantic-router Kustomization, its namespace pods, and all LLM-platform resources are gone from the cluster, so vmagent has no up{} target to scrape → the critical 'no live up samples' alert fires.
- **confidence:** 85%
- **provenance:** flux-system Kustomization sync introduced clusters/mycluster-0/llm-platform.yaml (new file) with spec.suspend: true, prune: true

## Symptom

vllm-semantic-router absent — new llm-platform umbrella Kustomization shipped suspended, blocking all LLM child resources

Affected resource: Kustomization flux-system/vllm-semantic-router

## Investigate

- what_changed(flux-system) shows the flux-system Kustomization synced a NEW file clusters/mycluster-0/llm-platform.yaml. The diff's own comments state: 'Default: spec.suspend = true → none of the LLM-specific Flux Kustomizations under clusters/mycluster-0-llm-platform/ are created, so no LLM resources land in the cluster (no vllm-semantic-router, no NVIDIA device plugin, no GPU NodePool, no LLM apps, no LLM EPI).'
- gitops_resource_status(Kustomization/vllm-semantic-router in flux-system) → NOT FOUND. gitops_tree → NOT FOUND (root). The workload's Flux object genuinely does not exist.
- pod_status(vllm-semantic-router) → 'no pods in namespace'. kube_events(vllm-semantic-router) → 'no Warning events'. The namespace is empty — no pods, no events, no activity.
- gitops_resource_status(Kustomization/llm-platform) → Ready=Unknown (suspended state). gitops_tree shows its source GitRepository/flux-system is Ready=True — the source is fine; the suspension is the gate.
- The umbrella's diff documents the intended enable path: 'flux resume kustomization llm-platform -n flux-system', and the teardown path references 'flux delete kustomization ... vllm-semantic-router' — confirming vllm-semantic-router is a direct child gated by this umbrella.
- controller_logs(kustomize-controller, resource=vllm-semantic-router, 220m) → no matching log lines — Flux never reconciled vllm-semantic-router because the suspended umbrella never created it.
- The OCIRepository llm/vllm-semantic-router source IS present and unchanged in flux-sources reconcile output — the artifact source exists, but nothing consumes it because the consumer Kustomization is never created under the suspended umbrella.

## Cause

1. **A new Flux 'llm-platform' umbrella Kustomization was committed to the cluster's flux-system sync with spec.suspend: true and prune: true. This is an intentional opt-in gate: while suspended, Flux does NOT create any of the child Kustomizations under clusters/mycluster-0-llm-platform/ — explicitly including vllm-semantic-router. With prune: true, Flux actively removes any previously-applied children. The result: the vllm-semantic-router Kustomization, its namespace pods, and all LLM-platform resources are gone from the cluster, so vmagent has no up{} target to scrape → the critical 'no live up samples' alert fires.** (85%) — change: flux-system Kustomization sync introduced clusters/mycluster-0/llm-platform.yaml (new file) with spec.suspend: true, prune: true

## Resolution

- If the LLM platform is meant to be live on this cluster, resume the umbrella so Flux creates (or recreates) the child Kustomizations including vllm-semantic-router: 'flux resume kustomization llm-platform -n flux-system' then 'flux reconcile kustomization llm-platform --with-source'. If the platform is NOT meant to be live here, the alert is firing against an intentionally-disabled component and should be scoped/suppressed for this cluster. (reversible=true)

## Unresolved

- Whether the LLM platform is intended to be enabled on this specific cluster (mycluster-0) or whether this cluster is meant to remain opt-out — only the platform/cluster owners can confirm intent. If opt-out is intended, the alert is mis-scoped, not a real outage.

## Citations

[1] flux-system Kustomization sync introduced clusters/mycluster-0/llm-platform.yaml (new file) with spec.suspend: true, prune: true

