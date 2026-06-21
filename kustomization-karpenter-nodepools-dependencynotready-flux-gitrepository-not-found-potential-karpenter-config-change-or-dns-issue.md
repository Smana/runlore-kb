---
type: Incident
title: 'Kustomization/karpenter-nodepools DependencyNotReady: Flux GitRepository not found, potential Karpenter config change or DNS issue.'
description: 'GitOps Repository Not Found: The flux-system/karpenter Kustomization depends on the flux-system/karpenter GitRepository, but the what_changed tool indicates that the GitRepositories (infra-artifact and apps-artifact) are not found. This directly prevents Flux from reconciling karpenter, leading to its DependencyNotReady state.'
tags:
    - runlore
    - incident
---

## Summary

Confidence 80%.

## Root causes
1. **GitOps Repository Not Found: The flux-system/karpenter Kustomization depends on the flux-system/karpenter GitRepository, but the what_changed tool indicates that the GitRepositories (infra-artifact and apps-artifact) are not found. This directly prevents Flux from reconciling karpenter, leading to its DependencyNotReady state.** (90%)
   - evidence: what_changed output: "error: get gitrepository flux-system/infra-artifact: gitrepositories.source.toolkit.fluxcd.io \"infra-artifact\" not found"
   - evidence: what_changed output: "error: get gitrepository flux-system/apps-artifact: gitrepositories.source.toolkit.fluxcd.io \"apps-artifact\" not found"
   - suggested: Investigate why flux-system/infra-artifact and flux-system/apps-artifact GitRepositories are not found. Verify the GitRepository definitions and connectivity to the Git provider. (reversible=true)
2. **Recent Karpenter Configuration Change: A recent commit "fix(karpenter): bump EC2NodeClass AMI alias to bot..." was stored as an artifact. This change might have introduced a misconfiguration or breaking change within Karpenter that is preventing it from becoming ready.** (60%)
   - evidence: Log entry: 2026-06-21T10:33:20Z stored artifact for commit 'fix(karpenter): bump EC2NodeClass AMI alias to bot...'
   - suggested: Investigate the details of the commit "fix(karpenter): bump EC2NodeClass AMI alias to bot..." and its impact on Karpenter's readiness. (reversible=true)
3. **Cluster DNS Resolution Failure: There are logs indicating DNS resolution failures for victoria-logs-victoria-logs-cluster-vlinsert.observability.svc.cluster.local. A broader DNS issue could be preventing Karpenter from resolving necessary internal or external services, contributing to its DependencyNotReady state.** (50%)
   - evidence: Log entry: Cannot send event error="Post \"http://victoria-logs-victoria-logs-cluster-vlinsert.observability.svc.cluster.local:9481/insert/loki/api/v1/push\": dial tcp: lookup victoria-logs-victoria-logs-cluster-vlinsert.observability.svc.cluster.local on 172.20.0.10:53: no such host"
   - suggested: Investigate the health and configuration of the cluster's DNS service. (reversible=true)

