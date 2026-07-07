---
type: Incident
title: vllm-semantic-router is down after being unintentionally deleted by a new, suspended Flux Kustomization
description: The vllm-semantic-router workload was unintentionally deleted by a new, suspended Flux 'umbrella' Kustomization.
resource: vllm-semantic-router/vllm-semantic-router
tags:
    - runlore
    - incident
    - deployment
    - vllm-semantic-router
timestamp: "2026-07-07T07:00:27Z"
fingerprint: a287720a9fbf3c68fdc172ad729d87e8eaf2fc9271cc8b1ba11da1943364cfe8
confidence: 0.95
provenance:
    - 92d2ad88866bbced05d9e18b394d003b79bd323a
---

## Decision

- **why keep:** The vllm-semantic-router workload was unintentionally deleted by a new, suspended Flux 'umbrella' Kustomization.
- **confidence:** 95%
- **provenance:** 92d2ad88866bbced05d9e18b394d003b79bd323a

## Symptom

vllm-semantic-router is down after being unintentionally deleted by a new, suspended Flux Kustomization

Affected resource: Deployment vllm-semantic-router/vllm-semantic-router

## Investigate

- A cluster-wide 'what_changed' call revealed a new Flux Kustomization named 'llm-platform' was added to the 'flux-system' namespace.
- The diff shows this new Kustomization was created with 'spec.suspend: true'.
- A comment in the manifest for 'llm-platform' explicitly states that it manages all LLM resources, including 'vllm-semantic-router', and that suspending it will prevent them from being deployed.
- The 'pod_status' tool confirmed there are no pods in the 'vllm-semantic-router' namespace, which is consistent with the managing GitOps resource being suspended and garbage-collecting its children.
- Logs from 'kustomize-controller' show that the 'VMServiceScrape' for 'vllm-semantic-router' still exists, explaining why monitoring is still trying to scrape the (now absent) pods and firing an alert.

## Cause

1. **The vllm-semantic-router workload was unintentionally deleted by a new, suspended Flux 'umbrella' Kustomization.** (95%) — change: 92d2ad88866bbced05d9e18b394d003b79bd323a

## Resolution

- Resume the 'llm-platform' Kustomization in the 'flux-system' namespace using the command: `flux resume kustomization llm-platform -n flux-system` (reversible=true)

## Citations

[1] 92d2ad88866bbced05d9e18b394d003b79bd323a

