---
type: Incident
title: vmalert ServiceDown — node 10.0.9.21 CPU saturation (37 pods on 4 cores) causes probe timeouts; Karpenter cannot…
description: 'Node ip-10-0-9-21.eu-west-3.compute.internal is CPU-saturated (load1=39.89 spiking to 54.51 on a 4-core node; CPU utilization 99.6-99.7%), causing ALL pods on that node — including vmalert — to fail health probes with ''context deadline exceeded''. vmalert''s liveness probe failed 91+ times, triggering container restarts (5→7 restarts, last at 19:05:24Z) and the ServiceDown alert. The node carries 37 pods (highest density in cluster) including CPU-heavy workloads from other namespaces: kyverno-admission-controller (780-1537m, up to 1.5 cores), crossplane (238-493m), provider-aws-s3 (142-320m), vmsingle (176-205m), cilium (162-190m).'
resource: observability/vmalert-victoria-metrics-k8s-stack
tags:
    - runlore
    - incident
    - pod
    - observability
timestamp: "2026-07-16T19:13:51Z"
fingerprint: d2081923c2a354af6504daa72712c155af612972b2f8cc8f295fbb2a5d1d72f0
confidence: 0.85
---

## Decision

- **why keep:** Node ip-10-0-9-21.eu-west-3.compute.internal is CPU-saturated (load1=39.89 spiking to 54.51 on a 4-core node; CPU utilization 99.6-99.7%), causing ALL pods on that node — including vmalert — to fail health probes with 'context deadline exceeded'. vmalert's liveness probe failed 91+ times, triggering container restarts (5→7 restarts, last at 19:05:24Z) and the ServiceDown alert. The node carries 37 pods (highest density in cluster) including CPU-heavy workloads from other namespaces: kyverno-admission-controller (780-1537m, up to 1.5 cores), crossplane (238-493m), provider-aws-s3 (142-320m), vmsingle (176-205m), cilium (162-190m).
- **confidence:** 85%

## Symptom

vmalert ServiceDown — node 10.0.9.21 CPU saturation (37 pods on 4 cores) causes probe timeouts; Karpenter cannot provision new nodes to relieve pressure

Affected resource: Pod observability/vmalert-victoria-metrics-k8s-stack-5f94b5f9c8-qcjxm

## Investigate

- node_load1 on 10.0.9.21: first=21.3 last=39.89 max=54.51, while all other nodes have load1 of 0-2.3
- Node CPU non-idle on 10.0.9.21: 3.98 cores out of 3.92 allocatable (101%+), utilization 99.6-99.7%
- kube_events: vmalert readiness probe failed x114, liveness probe failed x91 — all 'context deadline exceeded' on http://100.64.37.44:8080/health
- kube_events: simultaneously grafana (x90/x24), node-exporter (x30/x29), kube-state-metrics (x27/x13), vmsingle also fail probes on same node
- vmalert container restarts: 5→7, last restart at 19:05:24Z (pod_status shows last-exit=2026-07-16T19:05:02Z matching alert)
- Per-pod CPU on 10.0.9.21: kyverno-admission-controller 779-1537m, crossplane 238-493m, provider-aws-s3 142-320m, vmsingle 176-205m, cilium 162-190m — vmalert itself only 28-42m
- Node 10.0.9.21 has 37 running pods (highest in cluster; next is 42 on 10.0.12.93 which has lower load)
- Network drops are 0 on all nodes — not a network issue
- cloud_resource_health: 'Karpenter nodepool default: instances=8 spot=8 terminated=3'
- incident_timeline: FailedScheduling events at 17:51 — '0/3 nodes available: 1 untolerated taint(s), 2 Insufficient cpu. no new claims to deallocate'
- incident_timeline: more FailedScheduling at 17:51 — '0/4 nodes available: 2 Insufficient cpu, 2 untolerated taint(s). no new claims'
- kube_events (karpenter ns): karpenter pod readiness probe failed at 18:00:48Z — 'context deadline exceeded'
- kb_search returned karpenter-ec2nodeclass-ami-not-found runbook: 'Karpenter provisions no nodes... Pods sit Pending on Insufficient cpu/memory and the cluster cannot scale'

## Cause

1. **Node ip-10-0-9-21.eu-west-3.compute.internal is CPU-saturated (load1=39.89 spiking to 54.51 on a 4-core node; CPU utilization 99.6-99.7%), causing ALL pods on that node — including vmalert — to fail health probes with 'context deadline exceeded'. vmalert's liveness probe failed 91+ times, triggering container restarts (5→7 restarts, last at 19:05:24Z) and the ServiceDown alert. The node carries 37 pods (highest density in cluster) including CPU-heavy workloads from other namespaces: kyverno-admission-controller (780-1537m, up to 1.5 cores), crossplane (238-493m), provider-aws-s3 (142-320m), vmsingle (176-205m), cilium (162-190m).** (85%)
2. **Karpenter cannot provision new nodes to relieve the CPU pressure — it has terminated 3 nodes but FailedScheduling events show 'no new claims' and 'Insufficient cpu'. This prevents the cluster from self-healing by redistributing pods. The KB article 'karpenter-ec2nodeclass-ami-not-found' describes a matching pattern (EC2NodeClass AMI alias not found → NodePool not Ready → no NodeClaims created → pods stuck Pending). Karpenter's own pod also had a readiness probe failure at 18:00:48Z.** (30%)

## Resolution

- Drain node ip-10-0-9-21 (kubectl drain ip-10-0-9-21.eu-west-3.compute.internal --ignore-daemonsets --delete-emptydir-data) to force rescheduling of pods onto other nodes, OR cordon it to prevent new pods and gradually migrate. Consider adding nodeSelector/affinity or resource requests to prevent kyverno + crossplane + observability stack from co-locating on a single 4-core node. (reversible=true)
- Check Karpenter EC2NodeClass status (kubectl get ec2nodeclass) and NodePool readiness. If EC2NodeClass is Ready=Unknown with 'failed to discover any AMIs for alias', bump the amiSelectorTerms alias to a valid Bottlerocket version for the cluster's K8s version (or use bottlerocket@latest), then reconcile. This is the likely root cause of the cluster-wide capacity shortage. (reversible=true)

## Unresolved

- Could not directly verify the Karpenter EC2NodeClass status (kubectl get ec2nodeclass) — the runbook strongly suggests this is the cause of 'no new claims', but confirmation requires checking the EC2NodeClass Ready condition and the nodeclass-controller logs
- Could not determine why node 10.0.9.21 accumulated 37 pods — whether 3 node terminations caused mass rescheduling onto this node, or whether kube-scheduler packed it too densely from the start

