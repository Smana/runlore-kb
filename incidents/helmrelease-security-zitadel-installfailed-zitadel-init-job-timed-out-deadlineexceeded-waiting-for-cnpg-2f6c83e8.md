---
type: Incident
title: HelmRelease security/zitadel InstallFailed — zitadel-init Job timed out (DeadlineExceeded) waiting for CNPG…
description: 'The zitadel-init pre-install Job timed out with DeadlineExceeded (18:41:47Z) because it could not resolve the DNS name of its database (xplane-zitadel-cnpg-cluster-rw) — the pod log shows 583 retries of ''lookup xplane-zitadel-cnpg-cluster-rw on 172.20.0.10:53: no such host'' from 18:31:21Z to 18:41:45Z. The CNPG cluster''s Service/DNS never came into existence because the cluster itself never initialized. Helm''s pre-install hook waits for the Job, which waits for the database; with no database, both stall and the Helm install fails.'
resource: security/zitadel
tags:
    - runlore
    - incident
    - helmrelease
    - security
timestamp: "2026-08-23T19:40:14Z"
fingerprint: 2f6c83e8184a646a1b381f572dc7709cf4c27dec19ce727a438b1cfdcf3abd87
confidence: 0.8
provenance:
    - 'no GitOps change detected (what_changed: ''no changes found for the given selector'')'
    - 'No GitOps change (what_changed: no changes found). Karpenter provisioned a new spot node (ip-10-0-44-217) at ~18:43Z; this node lacked EBS CSI topology labels, triggering the PVC provisioning failure.'
---

## Decision

- **why keep:** The zitadel-init pre-install Job timed out with DeadlineExceeded (18:41:47Z) because it could not resolve the DNS name of its database (xplane-zitadel-cnpg-cluster-rw) — the pod log shows 583 retries of 'lookup xplane-zitadel-cnpg-cluster-rw on 172.20.0.10:53: no such host' from 18:31:21Z to 18:41:45Z. The CNPG cluster's Service/DNS never came into existence because the cluster itself never initialized. Helm's pre-install hook waits for the Job, which waits for the database; with no database, both stall and the Helm install fails.
- **confidence:** 80%
- **provenance:** no GitOps change detected (what_changed: 'no changes found for the given selector'), No GitOps change (what_changed: no changes found). Karpenter provisioned a new spot node (ip-10-0-44-217) at ~18:43Z; this node lacked EBS CSI topology labels, triggering the PVC provisioning failure.

## Symptom

HelmRelease security/zitadel InstallFailed — zitadel-init Job timed out (DeadlineExceeded) waiting for CNPG database cluster that never came up due to EBS PVC topology-key provisioning failure on a Karpenter node

Affected resource: HelmRelease security/zitadel

## Investigate

- kube_events: 18:41:47Z Warning Job/zitadel-init DeadlineExceeded: Job was active longer than specified deadline
- query_logs (zitadel-init container): 583× 'retrying initial database connection in a second: failed to connect to user=postgres database=postgres: hostname resolving error: lookup xplane-zitadel-cnpg-cluster-rw on 172.20.0.10:53: no such host' from 18:31:21Z→18:41:45Z
- gitops_resource_status HelmRelease security/zitadel: InstallFailed 'failed pre-install: failed early due to stalled resources: [Job/security/zitadel-init status: Failed]'
- Helm logs show the Job was created at 18:31:17Z and DeadlineExceeded at 18:36:18Z (first attempt) and 18:41:47Z (second attempt)
- kube_events: 18:44:17Z Warning PVC/xplane-zitadel-cnpg-cluster-1 ProvisioningFailed (×7): error generating accessibility requirements: no topology key found for node ip-10-0-44-217.eu-west-3.compute.internal
- kube_events: 18:42:28Z Warning Pod/xplane-zitadel-cnpg-cluster-1-full-recovery-6vtcd FailedScheduling: 0/4 nodes available: 1 Insufficient memory, 3 Insufficient cpu
- kube_events: 18:43:11Z FailedScheduling (×6): 0/5 nodes available: 1 Insufficient memory, 1 untolerated taint, 3 Insufficient cpu
- kube_events: 18:58:49Z Warning Job/xplane-zitadel-cnpg-cluster-1-full-recovery BackoffLimitExceeded
- query_logs (full-recovery): 6× 'Error while restoring a backup' and 6× 'restore error' from 18:45:59Z→18:58:46Z across 6 pods
- pod_status: 6× full-recovery pods all Failed with Error (exit 1), all on node ip-10-0-44-217
- cloud_resource_health: Karpenter nodepool default: instances=3 spot=3 terminated=1 — a spot node was terminated during the incident window
- kube_events: PVC provisioning eventually succeeded at 18:45:23Z (ProvisioningSucceeded) but the full-recovery Job pods still failed with restore errors
- metrics: kube_node_status_allocatable shows node ip-10-0-44-217 appeared at 18:43:53Z (new Karpenter node) with 3.92 CPU cores, 6.58GB memory, 100 pods — it was a smaller spot instance

## Cause

1. **The zitadel-init pre-install Job timed out with DeadlineExceeded (18:41:47Z) because it could not resolve the DNS name of its database (xplane-zitadel-cnpg-cluster-rw) — the pod log shows 583 retries of 'lookup xplane-zitadel-cnpg-cluster-rw on 172.20.0.10:53: no such host' from 18:31:21Z to 18:41:45Z. The CNPG cluster's Service/DNS never came into existence because the cluster itself never initialized. Helm's pre-install hook waits for the Job, which waits for the database; with no database, both stall and the Helm install fails.** (80%) — change: no GitOps change detected (what_changed: 'no changes found for the given selector')
2. **The CNPG cluster xplane-zitadel-cnpg-cluster-1 never initialized because its full-recovery Job failed on all 6+ attempts with 'Error while restoring a backup' / 'restore error'. The job exhausted its BackoffLimit at 18:58:49Z. The root of the recovery failure is an EBS PVC provisioning issue: 7× ProvisioningFailed events at 18:44:17Z stating 'error generating accessibility requirements: no topology key found for node ip-10-0-44-217.eu-west-3.compute.internal'. The EBS CSI driver could not determine the AZ topology for this Karpenter-provisioned spot node, so the PVC could not bind, and the full-recovery pod could not restore the backup. Additionally, initial scheduling failed with 'Insufficient cpu/memory' (18:42:28Z and 18:43:11Z) until a new node (ip-10-0-44-217) was provisioned by Karpenter — but that node lacked the topology key the EBS CSI driver needed.** (35%) — change: No GitOps change (what_changed: no changes found). Karpenter provisioned a new spot node (ip-10-0-44-217) at ~18:43Z; this node lacked EBS CSI topology labels, triggering the PVC provisioning failure.

## Resolution

- Fix the CNPG cluster initialization (see root cause #2), then reset the HelmRelease failure counters with 'flux -n security reconcile helmrelease zitadel --reset' so Flux performs a fresh install attempt. (reversible=true)
- 1) Investigate why the EBS CSI driver could not find the topology key for node ip-10-0-44-217 — check if the node has the required topology labels (topology.ebs.csi.aws.com/zone). 2) If the CNPG cluster has a backup/restore configuration, verify the backup is accessible and the recovery parameters are correct. 3) Consider pinning the CNPG cluster to on-demand nodes or adding node affinity for nodes with proper EBS CSI topology labels. 4) After fixing the CNPG cluster, reset the zitadel HelmRelease with 'flux -n security reconcile helmrelease zitadel --reset'. (reversible=true)

## Unresolved

- Why the EBS CSI driver reported 'no topology key found for node ip-10-0-44-217' — the node's labels (visible in the full-recovery pod's node_labels) DO include topology.ebs.csi.aws.com/zone, topology.kubernetes.io/zone, and topology.k8s.aws/zone-id, yet the PVC provisioning failed with this error 7 times before eventually succeeding at 18:45:23Z. This may be a transient EBS CSI driver issue or a timing problem with the node registration. A human should verify whether this is a recurring Karpenter+EBS CSI integration issue.
- Why the CNPG full-recovery Job's backup restore failed even after the PVC was successfully provisioned — the logs show 'Restore through plugin detected, proceeding...' immediately followed by 'Error while restoring a backup' and 'restore error'. The detailed error from the backup/restore plugin (e.g. barman-cloud) was not captured in the logs available; pod_logs returned RBAC forbidden for these pods. A human should check the CNPG cluster's backup configuration and the actual restore error in the CNPG operator logs.

## Citations

[1] no GitOps change detected (what_changed: 'no changes found for the given selector')
[2] No GitOps change (what_changed: no changes found). Karpenter provisioned a new spot node (ip-10-0-44-217) at ~18:43Z; this node lacked EBS CSI topology labels, triggering the PVC provisioning failure.

