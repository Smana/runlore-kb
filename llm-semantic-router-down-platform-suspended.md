---
type: Playbook
title: LLMPlatformSemanticRouterDown fires on an LLM-free cluster — the opt-in LLM platform is suspended
description: The critical ai-fleet alert LLMPlatformSemanticRouterDown (and LLMPlatformFleetAbortRateHigh) fires because the self-hosted LLM platform is opt-in and suspended by default, so its scrape target never exists. Expected on any cluster where the platform was not explicitly enabled — not an outage.
resource: vmrule://ai
tags: [victoriametrics, vmalert, ai, llm, vllm-semantic-router, false-positive, opt-in, suspended, flux, observability]
timestamp: 2026-07-16T00:00:00Z
---

# Symptom

- `LLMPlatformSemanticRouterDown` (`critical`, alertgroup `ai-fleet`) fires, sometimes alongside
  `LLMPlatformFleetAbortRateHigh`.
- Expression: `absent(up{job="vllm-semantic-router"}) == 1 or sum(up{job="vllm-semantic-router"}) == 0`.
- No vLLM / semantic-router / AI-gateway workloads are running on the cluster.

# Investigate

1. Check whether the LLM platform is actually deployed. It is **opt-in with two independent gates**
   (see the platform's `CLAUDE.md` "Self-Hosted LLM Platform"): an OpenTofu stack tagged `opt-in`,
   and a Flux umbrella Kustomization `llm-platform` with `spec.suspend: true` **by default**.
   ```
   kubectl get kustomization -n flux-system llm-platform -o jsonpath='{.spec.suspend}'   # true → platform off
   kubectl get pods -A | grep -iE 'vllm|semantic-router'                                  # → none
   ```
2. If the platform is intentionally off, the scrape target legitimately does not exist, so the
   `absent()` alert is a false positive — not a regression.

# Root cause

The `ai-fleet` VMRule (`ai`) shipped from **always-on** observability
(`observability/base/victoria-metrics-k8s-stack/vmrules/ai.yaml`), but the platform it monitors is
opt-in and suspended by default. An always-deployed `absent(up{job="vllm-semantic-router"})` rule
therefore fires on every cluster that has not explicitly enabled the LLM platform.

# Resolution

- Move the `ai-fleet` VMRule into the **platform-gated** tree so it deploys only when the platform is
  resumed: `apps/base/ai/llm/vmrule-ai-fleet.yaml`, added to `apps/base/ai/llm/kustomization.yaml`
  (that path is synced only by the suspended `apps-llm` Flux Kustomization, alongside
  `vmrule-llm-slo.yaml`), and removed from the always-on `vmrules/kustomization.yaml`.
- If it is firing right now, the always-on `ai` VMRule is pruned by that same commit once Flux
  reconciles the observability Kustomization; confirm with the verification below.
- Do **not** "fix" this by widening the alert or muting it globally — the correct state is that the
  rule simply is not present when the platform is off, and returns (still meaningful) when it is on.

# Verification

- `kubectl get vmrule -n observability ai` → `NotFound`.
- `ALERTS{alertname=~"LLMPlatform.*"}` clears after VMAlert reload.
- When the LLM platform *is* resumed (`flux resume kustomization llm-platform -n flux-system`), the
  rule reappears in the `llm` namespace and once again alerts if the router is truly down.

# Citations

- Platform `CLAUDE.md` — "Self-Hosted LLM Platform" (two opt-in gates: OpenTofu + suspended Flux umbrella).
- Sibling precedent: `apps/base/ai/llm/vmrule-llm-slo.yaml` already lives in the gated tree.
