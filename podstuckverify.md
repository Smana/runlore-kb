---
type: Incident
title: PodStuckVerify
description: The root cause of the incident is IP address exhaustion on the Kubernetes node where the `victoria-logs-vector-pktsw` pod is attempting to schedule. The Cilium CNI has no available IPs in its pool for that specific node, which prevents the pod sandbox from being created and the pod from starting.
tags:
    - runlore
    - incident
---

## Summary

Confidence 95%.

## Root causes
1. **The root cause of the incident is IP address exhaustion on the Kubernetes node where the `victoria-logs-vector-pktsw` pod is attempting to schedule. The Cilium CNI has no available IPs in its pool for that specific node, which prevents the pod sandbox from being created and the pod from starting.** (95%)
   - evidence: The pod `victoria-logs-vector-pktsw` is in a `Pending` state.
   - evidence: Kubernetes events for the pod show a `FailedCreatePodSandBox` error with the message: `no IPs currently available on the node, allocation will be retried once Cilium Operator allocates more IPs`.
   - evidence: No relevant GitOps or cloud infrastructure changes were found that would explain the issue.
   - evidence: Logs for Cilium components show no errors, suggesting this is a capacity issue rather than a CNI failure.
   - suggested: The immediate remediation is to add a new node to the cluster. This will provide a new node with a fresh IP address pool, allowing the pending pod to be scheduled. For a long-term fix, the cluster's IPAM strategy should be reviewed to ensure nodes have appropriately sized IP pools for the expected pod density. (reversible=false)

## Unresolved
- The PromQL query `cilium_ipam_ips{ip_state="in-use"} / cilium_ipam_ips{ip_state="total"} > 0.9` returned no data, so I could not metrically confirm which node was exhausted or the overall IPAM utilization across the cluster.

