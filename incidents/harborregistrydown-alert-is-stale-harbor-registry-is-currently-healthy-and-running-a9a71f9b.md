---
type: Incident
title: HarborRegistryDown alert is stale — Harbor registry is currently healthy and running
description: 'STALE/FALSE ALERT: The HarborRegistryDown alert fired at 09:20Z, but Harbor recovered from a transient cluster-wide capacity crisis over 2 hours earlier (~07:04Z) and is currently fully healthy. The alert''s claimed root cause (Crossplane AccessKey hitting AWS IAM AccessKeysPerUser quota) is NOT supported by live evidence.'
resource: tooling/harbor-registry
tags:
    - runlore
    - incident
    - deployment
    - tooling
timestamp: "2026-07-07T07:49:02Z"
fingerprint: a9a71f9b6fb8dd5c7b2bd6d89d051c76fbaa49b64a41005d3adc839a2786a126
confidence: 0.85
---

## Decision

- **why keep:** STALE/FALSE ALERT: The HarborRegistryDown alert fired at 09:20Z, but Harbor recovered from a transient cluster-wide capacity crisis over 2 hours earlier (~07:04Z) and is currently fully healthy. The alert's claimed root cause (Crossplane AccessKey hitting AWS IAM AccessKeysPerUser quota) is NOT supported by live evidence.
- **confidence:** 85%

## Symptom

HarborRegistryDown alert is stale — Harbor registry is currently healthy and running

Affected resource: Deployment tooling/harbor-registry

## Investigate

- pod_status: All 7 Harbor pods are Running and Ready — harbor-registry-746b9f5dfc-787jg (2/2), harbor-core-6b6db7984c-pxcm4 (1/1), harbor-jobservice-79db77b8bc-cdvdx (1/1), harbor-nginx-6864f4d6d8-8gff9 (1/1), harbor-portal-f6d4588b4-tk98q (1/1), harbor-exporter-545f5d7c98-dd5bg (1/1), harbor-trivy-0 (1/1). No CreateContainerConfigError, CrashLoopBackOff, or Pending pods.
- HelmRelease tooling/harbor is Ready=True (InstallSucceeded) — 'Helm install succeeded for release tooling/harbor.v1 with chart harbor@1.18.3'.
- kube_events: The last Warning event for any Harbor pod was at 07:04:56Z (BackOff on harbor-jobservice), which immediately resolved — the pod Started at 07:04:57Z. No Warning events after 07:04Z.
- kube_events: The AccessKey/xplane-harbor event at 06:53:00Z shows 'CannotResolveResourceReferences: cannot resolve references: mg.Spec.ForProvider.User: no resources matched selector' — a reference resolution error, NOT the 'LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2' claimed by the alert.
- The alert fired at 09:20Z, over 2 hours after Harbor pods came up healthy at 07:04:36-07:04:57Z.
- kube_events: Multiple FailedCreatePodSandBox events from 06:53-06:58Z with 'cilium-cni failed: unable to allocate IP via local cilium agent: no IPs currently available on the node' across harbor-core, harbor-registry, harbor-jobservice, harbor-portal pods.
- kube_events: FailedScheduling events for harbor-portal: '0/5 nodes are available: 1 Insufficient memory, 2 Insufficient cpu, 2 node(s) had untolerated taint(s)'.
- HelmRelease events: First install attempt at 06:53:18 timed out; second at 06:58:33 also timed out; third succeeded (InstallSucceeded) — pods created at 07:04:36.
- cloud_resource_health: EKS nodegroup main-2026070706255868320000000f is ACTIVE with no current issues — capacity was restored.
- what_changed: No GitOps changes to tooling/harbor-registry — the issue was infrastructure-side capacity, not a code/config change.

## Cause

1. **STALE/FALSE ALERT: The HarborRegistryDown alert fired at 09:20Z, but Harbor recovered from a transient cluster-wide capacity crisis over 2 hours earlier (~07:04Z) and is currently fully healthy. The alert's claimed root cause (Crossplane AccessKey hitting AWS IAM AccessKeysPerUser quota) is NOT supported by live evidence.** (85%)
2. **Underlying transient cause (now resolved): A cluster-wide capacity/Cilium IPAM exhaustion crisis from ~06:53-07:04Z prevented Harbor pods from scheduling and starting, causing the HelmRelease install to time out twice before succeeding on the third attempt.** (78%)

## Resolution

- Acknowledge and silence/close the stale HarborRegistryDown alert. Verify the alerting rule's evaluation window and auto-recovery logic to prevent future stale alerts. No action needed on Harbor itself — it is healthy. (reversible=false)
- No action needed — the capacity crisis self-resolved. Consider monitoring Cilium IPAM pool exhaustion and node autoscaling to prevent recurrence. (reversible=false)

## Unresolved

- Why the alert fired at 09:20Z when Harbor has been healthy since 07:04Z — the alerting rule's evaluation window, staleness threshold, or auto-recovery logic should be reviewed by the alerting/on-call team.
- Whether the AccessKey/xplane-harbor resource's reference resolution error (CannotResolveResourceReferences at 06:53Z) was ever resolved or is still pending — this is a Crossplane resource outside the scope of the available tools, but it does not appear to be impacting Harbor currently.

