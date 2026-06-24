---
type: Incident
title: PodStuckProfile
description: The cluster is out of allocatable pod IPs. The Cilium CNI is unable to acquire new IP addresses, which is preventing new pods, such as `victoria-logs-vector-pktsw`, from being scheduled.
tags:
    - runlore
    - incident
---

## Summary

Confidence 90%.

## Root causes
1. **The cluster is out of allocatable pod IPs. The Cilium CNI is unable to acquire new IP addresses, which is preventing new pods, such as `victoria-logs-vector-pktsw`, from being scheduled.** (90%)
   - evidence: The pod `victoria-logs-vector-pktsw` in the `observability` namespace is stuck in a `Pending` state.
   - evidence: Kubernetes events for the pod show the error: `Failed to create pod sandbox: ... unable to allocate IP via local cilium agent: ... "no IPs currently available on the node..."`.
   - evidence: The metric `cilium_operator_ipam_available_ips` has a value of 7, indicating a critical shortage of available IPs across the cluster.

## Unresolved
- The root cause of the Cilium operator's inability to allocate more IPs is unknown. Further investigation of the cilium-operator's logs and configuration is required.
- The specific node(s) that are out of IPs are not identified, as the failing pod is stuck in Pending and has not been scheduled to a node.

