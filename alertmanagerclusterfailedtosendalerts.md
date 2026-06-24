---
type: Incident
title: AlertmanagerClusterFailedToSendAlerts
description: Observability components blocked by Kubernetes policies disallowing HostPath volumes and host namespaces. This is preventing core logging and monitoring agents (victoria-logs-vector, prometheus-node-exporter) from functioning.
resource: observability/vmalertmanager-victoria-metrics-k8s-stack-0
tags:
    - runlore
    - incident
fingerprint: 53c3c898672b6fd3e2d31cd61ea1eb47b500f0e2551a04ff628c05ac41647a74
---

## Decision

- **why keep:** Observability components blocked by Kubernetes policies disallowing HostPath volumes and host namespaces. This is preventing core logging and monitoring agents (victoria-logs-vector, prometheus-node-exporter) from functioning.
- **confidence:** 90%

## Symptom

AlertmanagerClusterFailedToSendAlerts

## Investigate

- kube_events: "PolicyViolation: policy disallow-host-path/host-path fail: validation error: HostPath volumes are forbidden"
- kube_events: "PolicyViolation: policy disallow-host-namespaces/host-namespaces fail: validation error: Sharing the host namespaces is disallowed"
- kube_events: "FailedCreatePodSandBox: Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox ... plugin type=\"cilium-cni\" failed (add): unable to create endpoint: Put \"http://localhost/v1/endpoint/cilium-local:0\": EOF"
- kube_events: "unable to allocate IP via local cilium agent: [POST /ipam][502] postIpamFailure \"no IPs currently available on the node, allocation will be retried once Cilium Operator allocates more IPs\""
- kube_events: "Pod/victoria-logs-victoria-logs-single-server-0 FailedScheduling: 0/10 nodes are available: 2 Insufficient memory, 4 node(s) had untolerated taint(s), 5 Insufficient cpu."

## Cause

1. **Observability components blocked by Kubernetes policies disallowing HostPath volumes and host namespaces. This is preventing core logging and monitoring agents (victoria-logs-vector, prometheus-node-exporter) from functioning.** (90%)
2. **Cilium CNI network failures are impacting victoria-logs-vector pods, preventing pod sandbox creation due to endpoint and IP allocation issues.** (80%)
3. **Resource exhaustion (Insufficient CPU/memory) and untolerated taints are causing scheduling failures for the victoria-logs-victoria-logs-single-server-0 pod, a critical logging component.** (80%)

## Resolution

- Review and modify the cluster's admission policies to allow the necessary HostPath volumes and host namespaces for the affected observability components, or reconfigure these components to not require them. (reversible=true)
- Investigate the health and configuration of the Cilium CNI, including the Cilium agent and operator logs. Check for IP address exhaustion or network connectivity issues to the Cilium agent. (reversible=true)
- Increase cluster capacity (nodes, memory, CPU) in the appropriate node groups, or review and adjust the resource requests for the victoria-logs-victoria-logs-single-server-0 pod. Examine node taints to ensure they are not inadvertently blocking this workload. (reversible=true)

## Unresolved

- The precise policy definition and enforcement mechanism responsible for disallowing HostPath and host namespaces.
- The underlying cause of the Cilium CNI issues (e.g., misconfiguration, resource exhaustion within Cilium components, a bug).
- The specific reason for the 503 status code from the readiness probe of victoria-logs-victoria-logs-single-server-0.

