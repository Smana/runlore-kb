---
type: Incident
title: 'HelmRelease victoria-logs InstallFailed: Cilium IPAM exhaustion on node ip-10-0-3-147 blocks DaemonSet pod + Flux…'
description: 'Cilium per-node IPAM pool exhaustion on node ip-10-0-3-147.eu-west-3.compute.internal prevents the victoria-logs-vector DaemonSet pod from obtaining a pod IP. The node''s IPAM capacity is 42 IPv4 addresses and all 42 are allocated (cilium_ipam_capacity=42, cilium_ip_addresses=42), confirmed by cilium_operator_ipam_nodes{category="at-capacity"}=1 (exactly one of 9 nodes at capacity). The Pending pod victoria-logs-vector-6gtf8 is scheduled to this node and cannot create its pod sandbox — kube_events shows 201 repetitions of ''Failed to create pod sandbox: ... cilium-cni failed (add): unable to allocate IP via local cilium agent: no IPs currently available on the node, allocation will be retried once Cilium Operator allocates more IPs''. Because the DaemonSet needs a pod on every node (9 Running + 1 Pending = desired 10, ready 9), it stays status ''InProgress'', so Helm''s --wait times out → HelmRelease InstallFailed.'
resource: observability/victoria-logs
tags:
    - runlore
    - incident
    - helmrelease
    - observability
timestamp: "2026-08-02T10:02:27Z"
fingerprint: 790c311646f2f1ad937e1024332cefc503c490c28211ddb66d67ae098ba0396f
confidence: 0.88
provenance:
    - No GitOps change caused this — what_changed shows no recent diff for observability; the node simply reached its Cilium per-node IPAM ceiling organically as pods accumulated.
    - helm-controller terminal error at 2026-08-02T09:37:55Z
---

## Decision

- **why keep:** Cilium per-node IPAM pool exhaustion on node ip-10-0-3-147.eu-west-3.compute.internal prevents the victoria-logs-vector DaemonSet pod from obtaining a pod IP. The node's IPAM capacity is 42 IPv4 addresses and all 42 are allocated (cilium_ipam_capacity=42, cilium_ip_addresses=42), confirmed by cilium_operator_ipam_nodes{category="at-capacity"}=1 (exactly one of 9 nodes at capacity). The Pending pod victoria-logs-vector-6gtf8 is scheduled to this node and cannot create its pod sandbox — kube_events shows 201 repetitions of 'Failed to create pod sandbox: ... cilium-cni failed (add): unable to allocate IP via local cilium agent: no IPs currently available on the node, allocation will be retried once Cilium Operator allocates more IPs'. Because the DaemonSet needs a pod on every node (9 Running + 1 Pending = desired 10, ready 9), it stays status 'InProgress', so Helm's --wait times out → HelmRelease InstallFailed.
- **confidence:** 88%
- **provenance:** No GitOps change caused this — what_changed shows no recent diff for observability; the node simply reached its Cilium per-node IPAM ceiling organically as pods accumulated., helm-controller terminal error at 2026-08-02T09:37:55Z

## Symptom

HelmRelease victoria-logs InstallFailed: Cilium IPAM exhaustion on node ip-10-0-3-147 blocks DaemonSet pod + Flux retries exhausted

Affected resource: HelmRelease observability/victoria-logs

## Investigate

- pod_status: victoria-logs-vector-6gtf8 Pending ready=0/1 on node=ip-10-0-3-147.eu-west-3.compute.internal (49m)
- kube_events: 'Pod/victoria-logs-vector-6gtf8 FailedCreatePodSandBox (x201): Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox ... cilium-cni failed (add): unable to allocate IP via local cilium agent: [POST /ipam][502] postIpamFailure "no IPs currently available on the node, allocation will be retried once Cilium Operator allocates more IPs"'
- query_metrics: cilium_ipam_capacity{node="ip-10-0-3-147..."} = 42 AND cilium_ip_addresses{node="ip-10-0-3-147...",family="ipv4"} = 42 → 100% utilization
- query_metrics: cilium_operator_ipam_nodes{category="at-capacity"} = 1, {category="total"} = 9 → exactly one node exhausted
- query_metrics_range: cilium_ip_addresses for this node flat at 42 across the full 120m window — the node was already saturated before the incident
- HelmRelease status: 'timeout waiting for: [DaemonSet/observability/victoria-logs-vector status: InProgress]' — Helm --wait cannot complete because 1 of 10 DaemonSet pods is Pending
- All other 9 victoria-logs-vector pods are Running ready=1/1 on other nodes — the failure is isolated to this one node's IPAM pool
- controller_logs (helm-controller): '2026-08-02T09:37:55.947Z Reconciler Error ... error: terminal error: exceeded maximum retries: cannot remediate failed release'
- controller_logs: '2026-08-02T09:37:55.925Z release is in a failed state' — the release storage is wedged
- HelmRelease events show the install→timeout→uninstall→reinstall cycle repeating: InstallFailed at 09:07:39Z, UninstallSucceeded at 09:07:41Z, then re-install at 09:07:55Z → InstallFailed again at 09:37:55Z
- kb_search runbook 'helmrelease-terminal-failed-exhausted-retries': 'once retries are exhausted Flux stops retrying even after root cause is fixed; flux reconcile helmrelease --reset clears the counters'

## Cause

1. **Cilium per-node IPAM pool exhaustion on node ip-10-0-3-147.eu-west-3.compute.internal prevents the victoria-logs-vector DaemonSet pod from obtaining a pod IP. The node's IPAM capacity is 42 IPv4 addresses and all 42 are allocated (cilium_ipam_capacity=42, cilium_ip_addresses=42), confirmed by cilium_operator_ipam_nodes{category="at-capacity"}=1 (exactly one of 9 nodes at capacity). The Pending pod victoria-logs-vector-6gtf8 is scheduled to this node and cannot create its pod sandbox — kube_events shows 201 repetitions of 'Failed to create pod sandbox: ... cilium-cni failed (add): unable to allocate IP via local cilium agent: no IPs currently available on the node, allocation will be retried once Cilium Operator allocates more IPs'. Because the DaemonSet needs a pod on every node (9 Running + 1 Pending = desired 10, ready 9), it stays status 'InProgress', so Helm's --wait times out → HelmRelease InstallFailed.** (88%) — change: No GitOps change caused this — what_changed shows no recent diff for observability; the node simply reached its Cilium per-node IPAM ceiling organically as pods accumulated.
2. **The Flux HelmRelease observability/victoria-logs has exhausted its install remediation retries and is now in a terminal failed state. helm-controller logs show 'terminal error: exceeded maximum retries: cannot remediate failed release' at 09:37:55Z. Even after the Cilium IPAM exhaustion is resolved, Flux will NOT retry the install on its own — the failure counters persist in status and a plain reconcile or suspend/resume will not clear them. The KB runbook 'helmrelease-terminal-failed-exhausted-retries' confirms: `flux reconcile helmrelease --reset` is required to reset the counters and trigger a fresh install attempt.** (45%) — change: helm-controller terminal error at 2026-08-02T09:37:55Z

## Resolution

- Free a Cilium IP on node ip-10-0-3-147 so the Pending victoria-logs-vector pod can get an IP. Options: (1) increase the Cilium IPAM pool size for this node via CiliumConfig/IPPool; (2) check for leaked/stale IPs by restarting the Cilium agent (cilium-fkkfd) on that node to trigger IP garbage collection; (3) drain a non-critical pod off this node to free an IP. The subnet-level pool is NOT exhausted (16000+ available per AZ), so this is a per-node allocation limit, not a VPC subnet issue. (reversible=true)
- After the Cilium IPAM issue is resolved (pod victoria-logs-vector-6gtf8 transitions to Running), reset the HelmRelease failure counters: `flux -n observability reconcile helmrelease victoria-logs --reset`. If the helm release storage is wedged (helm history shows only failed revisions), run `helm -n observability uninstall victoria-logs` first, then the --reset reconcile. (reversible=true)

## Unresolved

- Whether the 42-IP per-node Cilium IPAM capacity is the intended/configured limit or should be raised — this is a Cilium configuration question (CiliumConfig/IPPool spec) that requires checking the cluster's Cilium installation values, which are outside the observability namespace scope of this investigation.
- Whether there are leaked/stale Cilium IPs on node ip-10-0-3-147 (the node has 42 IPs allocated but only ~7 observability pods visible — other namespaces' pods account for the rest, but some could be stale; a Cilium agent restart would confirm via IP GC).

## Citations

[1] No GitOps change caused this — what_changed shows no recent diff for observability; the node simply reached its Cilium per-node IPAM ceiling organically as pods accumulated.
[2] helm-controller terminal error at 2026-08-02T09:37:55Z

