---
type: Incident
title: KubeAPIDown caused by misconfigured security policy
description: A new, restrictive set of Kyverno policies was applied to the cluster without excluding the 'kube-system' namespace. These policies are preventing the kube-apiserver and other critical control plane pods from starting, leading to the API server outage.
resource: kube-system/kube-apiserver
tags:
    - runlore
    - incident
timestamp: "2026-06-28T13:57:42Z"
fingerprint: 6be9f0cb6dd9cdfc7233e63e59f89a1792fec7ec026aa43000d2706f4b56d0d0
---

## Decision

- **why keep:** A new, restrictive set of Kyverno policies was applied to the cluster without excluding the 'kube-system' namespace. These policies are preventing the kube-apiserver and other critical control plane pods from starting, leading to the API server outage.
- **confidence:** 95%
- **provenance:** flux Kustomization/security (sync): ..4d05a569258d85e4c491e440f51e699691c4ae5c078b12aad0ca833cfad20d41

## Symptom

KubeAPIDown caused by misconfigured security policy

## Investigate

- Recent GitOps sync of a 'security' Kustomization.
- Massive number of 'PolicyViolation' events in the kube-system namespace, blocking pod creation.
- Kyverno logs confirm it is the source of the policy denials.
- Absence of kube-apiserver pods in the kube-system namespace.

## Cause

1. **A new, restrictive set of Kyverno policies was applied to the cluster without excluding the 'kube-system' namespace. These policies are preventing the kube-apiserver and other critical control plane pods from starting, leading to the API server outage.** (95%) — change: flux Kustomization/security (sync): ..4d05a569258d85e4c491e440f51e699691c4ae5c078b12aad0ca833cfad20d41

## Resolution

- Suspend the 'security' Kustomization in the 'flux-system' namespace. This will remove the policies and should allow the kube-apiserver to restart. (reversible=true)

