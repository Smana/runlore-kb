---
type: Incident
title: LLMPlatformSemanticRouterDown
description: The vllm-semantic-router workload is not deployed, as its corresponding Kustomization resource was not found. This results in no pods running for the workload.
resource: llm-platform/vllm-semantic-router
tags:
    - runlore
    - incident
timestamp: "2026-06-28T10:56:36Z"
fingerprint: 12057666a9aa6fd0b7f39c7e6305e7b5f0fcee262dbfc02f169ba72a5e62397e
---

## Decision

- **why keep:** The vllm-semantic-router workload is not deployed, as its corresponding Kustomization resource was not found. This results in no pods running for the workload.
- **confidence:** 90%

## Symptom

LLMPlatformSemanticRouterDown

## Investigate

- pod_status returned 'no pods in namespace "llm-platform" matching app=vllm-semantic-router'.
- gitops_resource_status for Kustomization vllm-semantic-router in llm-platform returned 'NOT FOUND'.
- what_changed returned 'no changes found for the given selector', indicating no recent GitOps activity related to this workload.

## Cause

1. **The vllm-semantic-router workload is not deployed, as its corresponding Kustomization resource was not found. This results in no pods running for the workload.** (90%)

## Resolution

- Investigate whether the vllm-semantic-router workload was intentionally undeployed or if there's a misconfiguration preventing its deployment. If it should be running, deploy the Kustomization resource for vllm-semantic-router in the llm-platform namespace. (reversible=true)

## Unresolved

- Whether the undeployment of vllm-semantic-router was intentional or an accidental configuration drift.

