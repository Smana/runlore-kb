---
type: Incident
title: LLMPlatformSemanticRouterDown — vllm-semantic-router never deployed; llm-platform Flux Kustomization is…
description: 'The llm-platform Flux Kustomization (flux-system/llm-platform) is suspended (spec.suspend: true), so its child Kustomizations — including vllm-semantic-router — were never created. No HelmRelease, no Deployment, no pods, no scrape targets, no up{} samples. The alert fires because vmagent has nothing to scrape for job=vllm-semantic-router.'
resource: flux-system/llm-platform
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-07-14T21:31:40Z"
fingerprint: 1d81186e8b4917f96bc0bd8603236f7092d43b6a622c943d140613d06be0276c
confidence: 0.82
provenance:
    - flux-system Kustomization sync at 2026-07-14T21:20:02Z — new file clusters/mycluster-0/llm-platform.yaml (+45/-0)
---

## Decision

- **why keep:** The llm-platform Flux Kustomization (flux-system/llm-platform) is suspended (spec.suspend: true), so its child Kustomizations — including vllm-semantic-router — were never created. No HelmRelease, no Deployment, no pods, no scrape targets, no up{} samples. The alert fires because vmagent has nothing to scrape for job=vllm-semantic-router.
- **confidence:** 82%
- **provenance:** flux-system Kustomization sync at 2026-07-14T21:20:02Z — new file clusters/mycluster-0/llm-platform.yaml (+45/-0)

## Symptom

LLMPlatformSemanticRouterDown — vllm-semantic-router never deployed; llm-platform Flux Kustomization is intentionally suspended

Affected resource: Kustomization flux-system/llm-platform

## Investigate

- what_changed(flux-system) shows a newly added file clusters/mycluster-0/llm-platform.yaml creating Kustomization flux-system/llm-platform with spec.suspend: true. The file's own comments state: 'Default: spec.suspend = true → none of the LLM-specific Flux Kustomizations ... are created, so no LLM resources land in the cluster (no vllm-semantic-router, no NVIDIA device plugin, no GPU NodePool, no LLM apps, no LLM EPI).'
- gitops_resource_status(Kustomization/llm-platform, flux-system) → Ready=Unknown (suspended state); sourceRef GitRepository/flux-system is Ready=True
- gitops_resource_status(HelmRelease/vllm-semantic-router, llm) → NOT FOUND — searched given namespace, flux-system, and all namespaces by name
- pod_status(vllm-semantic-router) → 'no pods in namespace'; pod_status(llm) → 'no pods in namespace'
- OCIRepository/llm/vllm-semantic-router is Ready=True (chart source oci://ghcr.io/vllm-project/charts/semantic-router v0.2.0) — the source exists but no HelmRelease consumes it
- helm-controller logs (210m) show many HelmReleases reconciling (cert-manager, crossplane, karpenter, tailscale, victoria-metrics, etc.) but ZERO entries for vllm-semantic-router
- kustomize-controller logs have no entries mentioning llm-platform — consistent with suspension (Flux does not reconcile suspended Kustomizations)
- query_metrics(up{job="vllm-semantic-router"}) → no series matched; discover_metrics(namespace=vllm-semantic-router) → no metrics at all in the last 360m
- ALERTS{alertname="LLMPlatformSemanticRouterDown"} → pending since 18:16Z, firing since 18:21Z, continuously 1 for the entire window
- The file comments explicitly document the enable command: 'flux resume kustomization llm-platform -n flux-system'

## Cause

1. **The llm-platform Flux Kustomization (flux-system/llm-platform) is suspended (spec.suspend: true), so its child Kustomizations — including vllm-semantic-router — were never created. No HelmRelease, no Deployment, no pods, no scrape targets, no up{} samples. The alert fires because vmagent has nothing to scrape for job=vllm-semantic-router.** (82%) — change: flux-system Kustomization sync at 2026-07-14T21:20:02Z — new file clusters/mycluster-0/llm-platform.yaml (+45/-0)

## Resolution

- Determine intent: (A) If the LLM platform is supposed to run on this cluster, run 'flux resume kustomization llm-platform -n flux-system' to create the child Kustomizations (including vllm-semantic-router) and deploy the workload. (B) If this cluster is not intended to run the LLM platform, suppress or re-scope the LLMPlatformSemanticRouterDown alert rule to only clusters where the platform is enabled. (reversible=true)

## Unresolved

- Whether the LLM platform is intended to be enabled on this specific cluster — the llm-platform.yaml was added with spec.suspend: true (opt-in default), but the alert's critical severity and message ('All inference traffic flows through this component — outage means the public LLM endpoint is broken') suggest it may be expected to be running. A human must confirm intent: should this cluster have the LLM platform enabled?
- Whether the alert rule LLMPlatformSemanticRouterDown should be scoped to only clusters where the LLM platform is enabled, or if it was intentionally deployed cluster-wide to catch exactly this kind of 'forgot to enable' situation

## Citations

[1] flux-system Kustomization sync at 2026-07-14T21:20:02Z — new file clusters/mycluster-0/llm-platform.yaml (+45/-0)

