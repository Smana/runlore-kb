---
type: Incident
title: HelmRelease harbor-valkey SourceNotReady — transient CPU capacity shortage delaying install (recovering)
description: The alert's 'SourceNotReady' message is stale/misleading — the HelmChart source (flux-system/tooling-harbor-valkey) is actually Ready=True (ChartPullSucceeded, pulled valkey chart v3.0.31). The HelmRelease is Progressing with a 5m install timeout, and the real blocker was a cluster-wide CPU capacity shortage that prevented the harbor-valkey StatefulSet pods from scheduling in time.
resource: tooling/harbor-valkey
tags:
    - runlore
    - incident
    - helmrelease
    - tooling
timestamp: "2026-07-14T19:28:51Z"
fingerprint: f27284f104369b81ba9c451923703df053c17a3b8180d43f9f8f114ca07d2ad1
confidence: 0.78
---

## Decision

- **why keep:** The alert's 'SourceNotReady' message is stale/misleading — the HelmChart source (flux-system/tooling-harbor-valkey) is actually Ready=True (ChartPullSucceeded, pulled valkey chart v3.0.31). The HelmRelease is Progressing with a 5m install timeout, and the real blocker was a cluster-wide CPU capacity shortage that prevented the harbor-valkey StatefulSet pods from scheduling in time.
- **confidence:** 78%

## Symptom

HelmRelease harbor-valkey SourceNotReady — transient CPU capacity shortage delaying install (recovering)

Affected resource: HelmRelease tooling/harbor-valkey

## Investigate

- gitops_resource_status HelmChart flux-system/tooling-harbor-valkey: Ready=True, ChartPullSucceeded at 19:23:23Z
- gitops_resource_status HelmRelease tooling/harbor-valkey: Ready=Unknown (Progressing), 'Running install action with timeout of 5m0s'
- kube_events: harbor-valkey-primary-0 FailedScheduling (x8): '0/7 nodes available: 3 untolerated taint(s), 4 Insufficient cpu' from 19:23:24 through 19:24:13
- kube_events: harbor-valkey-replicas-0 same FailedScheduling pattern, same window
- Multiple other pods in tooling namespace simultaneously Pending on Insufficient cpu (harbor-portal, harbor-trivy, headlamp) — cluster-wide, not valkey-specific

## Cause

1. **The alert's 'SourceNotReady' message is stale/misleading — the HelmChart source (flux-system/tooling-harbor-valkey) is actually Ready=True (ChartPullSucceeded, pulled valkey chart v3.0.31). The HelmRelease is Progressing with a 5m install timeout, and the real blocker was a cluster-wide CPU capacity shortage that prevented the harbor-valkey StatefulSet pods from scheduling in time.** (78%)

## Resolution

- The capacity shortage is self-resolving via Karpenter. Monitor the HelmRelease — if it completes within the 5m install window, no action needed. If it times out and goes terminal InstallFailed (exhausted retries), run: flux -n tooling reconcile helmrelease harbor-valkey --reset (reversible=true)

## Unresolved

- Whether the Karpenter EC2NodeClass AMI alias issue (per KB runbook) was the original cause of the capacity shortage, or whether it was simply slow node provisioning under high demand. The EC2NodeClass status could not be directly inspected with the available tools.
- Whether the harbor-valkey HelmRelease install will complete within its 5m timeout (started ~19:23:21, timeout ~19:28:21). If replicas-1 doesn't become Ready in time, the install may fail and require --reset.

