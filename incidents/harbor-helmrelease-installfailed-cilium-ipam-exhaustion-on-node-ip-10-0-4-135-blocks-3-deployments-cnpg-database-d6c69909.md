---
type: Incident
title: 'Harbor HelmRelease InstallFailed: Cilium IPAM exhaustion on node ip-10-0-4-135 blocks 3 Deployments; CNPG database…'
description: 'Cilium per-node IPAM exhaustion on node ip-10-0-4-135 (42/42 IPs used, 0 available for 2+ hours) prevents 3 Harbor Deployments (harbor-core, harbor-exporter, harbor-nginx) from obtaining pod IPs. Their pods are stuck Pending with FailedCreatePodSandBox, the Deployments never reach Ready (status InProgress), and Helm''s --wait times out → HelmRelease InstallFailed. This matches the KB runbook ''HelmRelease InstallFailed: a node at Cilium ENI IPAM capacity blocks a pod'' (100% confidence, verified 2026-08-02).'
resource: tooling/harbor
tags:
    - runlore
    - incident
    - helmrelease
    - tooling
timestamp: "2026-08-23T20:16:58Z"
fingerprint: d6c699096cf69d7310f3799636e57cde3879fdefb67741a92cffcd70856611fa
confidence: 0.85
provenance:
    - No GitOps change involved — node reached IPAM ceiling organically as pods accumulated
---

## Decision

- **why keep:** Cilium per-node IPAM exhaustion on node ip-10-0-4-135 (42/42 IPs used, 0 available for 2+ hours) prevents 3 Harbor Deployments (harbor-core, harbor-exporter, harbor-nginx) from obtaining pod IPs. Their pods are stuck Pending with FailedCreatePodSandBox, the Deployments never reach Ready (status InProgress), and Helm's --wait times out → HelmRelease InstallFailed. This matches the KB runbook 'HelmRelease InstallFailed: a node at Cilium ENI IPAM capacity blocks a pod' (100% confidence, verified 2026-08-02).
- **confidence:** 85%
- **provenance:** No GitOps change involved — node reached IPAM ceiling organically as pods accumulated

## Symptom

Harbor HelmRelease InstallFailed: Cilium IPAM exhaustion on node ip-10-0-4-135 blocks 3 Deployments; CNPG database recovery also failing

Affected resource: HelmRelease tooling/harbor

## Investigate

- pod_status: harbor-core-7f68c474f-5p97n Pending 0/1 on ip-10-0-4-135, harbor-exporter-6c4dc4c898-ddqdg Pending 0/1 on ip-10-0-4-135, harbor-nginx-8ff8cbcd5-v889g Pending 0/1 on ip-10-0-4-135
- kube_events: FailedCreatePodSandBox (x475/x473/x475) 'cilium-cni failed (add): unable to allocate IP via local cilium agent: [POST /ipam][502] postIpamFailure all CIDR ranges are exhausted' on all 3 pods
- query_metrics: cilium_ip_addresses{node=ip-10-0-4-135,family=ipv4}=42 and cilium_ipam_capacity{node=ip-10-0-4-135}=0 — node is at 100% IP utilization
- query_metrics_range: cilium_ip_addresses flat at 42 for entire 120m window (since 18:12); cilium_ipam_capacity flat at 0 (briefly hit 2 at 18:14 then back to 0)
- gitops_resource_status: HelmRelease tooling/harbor Ready=False InstallFailed, timeout waiting for harbor-jobservice, harbor-nginx, harbor-core, harbor-exporter status InProgress
- Other 3 Harbor components healthy: harbor-portal 1/1 Running, harbor-registry 2/2 Running, harbor-trivy-0 1/1 Running — failure isolated to pods on the exhausted node
- No GitOps changes to Harbor in the window (what_changed only shows old 2024 crds-actions-runner-controller sync)
- pod_status: harbor-jobservice-5b7cf89bfb-zjnjs Running 0/1 restarts=25 CrashLoopBackOff (exit 2) on ip-10-0-11-116
- query_logs: 'panic: failed to load configuration, error: failed to load rest config' and '[ERROR] Failed on load rest config err:Get http://harbor-core:80/api/v2.0/internalconfig: dial tcp 172.20.4.126:80: connect: connection refused' (x5 occurrences)
- kube_events: BackOff (x131) for harbor-jobservice crash-looping
- harbor-core is Pending 0/1 on ip-10-0-4-135 with no IP — the connection refused is because harbor-core has no pod IP to serve on
- This is a cascade: once harbor-core gets an IP and starts, jobservice should recover. But jobservice also needs harbor-core to be healthy (not just started).
- pod_status: 7 xplane-harbor-cnpg-cluster-1-full-recovery-* pods all Failed (exit 1) on ip-10-0-44-217 — a node with 23 free IPs, so NOT IPAM-related
- query_logs: 'Error while restoring a backup', 'restore error', 'WAL archive check failed for server xplane-harbor-cnpg-cluster: Expected empty archive' (x7 across 7 pods, first 18:45 → last 18:59)
- kube_events: 'SQLInstance/xplane-harbor ComposeResources: Composed resource xplane-harbor-cnpg-cluster is not yet ready' (x54) and 'xplane-harbor-cnpg-registry is not yet ready' (x60)
- kube_events: 'Backup/xplane-harbor-cnpg-daily-backup FindingPod: Couldn't find target pod xplane-harbor-cnpg-cluster-1' (x142) — no database pod exists
- No actual xplane-harbor-cnpg-cluster-1 pod in pod_status — only recovery jobs exist, cluster is in bootstrap/recovery state
- discover_log_fields: 252 log entries from CNPG recovery pods, all from bootstrap-controller/full-recovery/plugin-barman-cloud containers

## Cause

1. **Cilium per-node IPAM exhaustion on node ip-10-0-4-135 (42/42 IPs used, 0 available for 2+ hours) prevents 3 Harbor Deployments (harbor-core, harbor-exporter, harbor-nginx) from obtaining pod IPs. Their pods are stuck Pending with FailedCreatePodSandBox, the Deployments never reach Ready (status InProgress), and Helm's --wait times out → HelmRelease InstallFailed. This matches the KB runbook 'HelmRelease InstallFailed: a node at Cilium ENI IPAM capacity blocks a pod' (100% confidence, verified 2026-08-02).** (85%) — change: No GitOps change involved — node reached IPAM ceiling organically as pods accumulated
2. **harbor-jobservice CrashLoopBackOff (25 restarts, exit 2) is a direct cascade of harbor-core being unreachable. The jobservice pod has an IP (on node ip-10-0-11-116) but panics on startup because it cannot fetch its configuration from harbor-core:80 which has no pod IP (harbor-core is Pending on the IPAM-exhausted node).** (80%)
3. **CNPG PostgreSQL cluster (xplane-harbor-cnpg-cluster-1) is not running — 7 full-recovery Jobs have all Failed with 'Error while restoring a backup' and 'WAL archive check failed: Expected empty archive'. No primary database pod exists. This is an independent issue on a node with available IPAM capacity (ip-10-0-44-217), not caused by IPAM. Even after fixing IPAM, harbor-core and harbor-exporter (which depend on PostgreSQL) will not become Ready until the database is available. The Crossplane SQLInstance is still composing ('Composed resource xplane-harbor-cnpg-cluster is not yet ready').** (50%)

## Resolution

- Free IPs on node ip-10-0-4-135 by deleting replicated pods that have copies elsewhere (kubectl delete pod on a Deployment pod on that node — it reschedules to another node and releases an IP). Need to free at least 3 IPs for the 3 stuck Harbor pods. Alternatively, cordon the node and delete the 3 stuck Harbor pods so the scheduler places new pods on nodes with IPAM capacity (ip-10-0-44-217 has 23 free, ip-10-0-27-183 has 16 free). Then force-reconcile: flux reconcile hr harbor -n tooling --force. If that reports exhausted retries, escalate to flux reconcile hr harbor -n tooling --reset. (reversible=true)
- Will self-resolve once harbor-core gets an IP (from fixing root cause #1) and becomes Ready. If jobservice doesn't recover after harbor-core is up, restart it: kubectl rollout restart deploy harbor-jobservice -n tooling. (reversible=true)
- Investigate the CNPG recovery failure separately. The 'Expected empty archive' error suggests the WAL archive location may be empty or misconfigured. Check the CNPG Cluster resource status and the barman-cloud backup configuration. This needs database expertise to resolve — may require re-initializing the cluster without restore, or fixing the backup target. Once the database is up, harbor-core and harbor-exporter can become Ready (after IPAM is also fixed). (reversible=false)

## Unresolved

- Why is the CNPG cluster in a full-recovery/bootstrap state rather than running normally? Is this a fresh cluster rebuild, or did the primary database pod crash and trigger recovery? The recovery jobs fail with 'Expected empty archive' — a human should determine whether the backup target is correctly configured and whether the cluster should bootstrap fresh instead of restoring.
- Whether freeing IPs on node ip-10-0-4-135 or rescheduling pods to other nodes will be sufficient, or whether the CNPG database must be fixed first for harbor-core to become healthy. Both issues likely need resolution before the HelmRelease can succeed.

## Citations

[1] No GitOps change involved — node reached IPAM ceiling organically as pods accumulated

