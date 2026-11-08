---
type: Incident
title: Kustomization/zitadel HealthCheckFailed — CNPG cluster recovery blocked by non-empty S3 WAL archive (barman…
description: 'The CNPG cluster `xplane-zitadel-cnpg-cluster-1` cannot start because its backup recovery fails: the `barman-cloud-check-wal-archive` tool checks the S3 WAL archive destination and finds it is NOT empty (it contains pre-existing WAL files, likely from a previous cluster instance with the same name), producing the error `WAL archive check failed for server xplane-zitadel-cnpg-cluster: Expected empty archive`. This aborts the recovery job (6 failed pods, all exit 1), the CNPG cluster never reaches Ready, and Crossplane''s SQLInstance/xplane-zitadel keeps reporting ''Composed resource xplane-zitadel-cnpg-cluster is not yet ready''.'
resource: security/zitadel
alert_resource: flux-system/zitadel
tags:
    - runlore
    - incident
    - helmrelease
    - security
timestamp: "2026-08-23T20:09:40Z"
fingerprint: 5d199d5e5cc01ddf8896b745c6607e05cb549d2d2583752ece46df2d8b844cb1
confidence: 0.75
provenance:
    - No GitOps change detected — this is a data/state issue in the S3 WAL archive, not a config change. what_changed(security) and what_changed(flux-system/zitadel) returned 'no changes found'.
---

## Decision

- **why keep:** The CNPG cluster `xplane-zitadel-cnpg-cluster-1` cannot start because its backup recovery fails: the `barman-cloud-check-wal-archive` tool checks the S3 WAL archive destination and finds it is NOT empty (it contains pre-existing WAL files, likely from a previous cluster instance with the same name), producing the error `WAL archive check failed for server xplane-zitadel-cnpg-cluster: Expected empty archive`. This aborts the recovery job (6 failed pods, all exit 1), the CNPG cluster never reaches Ready, and Crossplane's SQLInstance/xplane-zitadel keeps reporting 'Composed resource xplane-zitadel-cnpg-cluster is not yet ready'.
- **confidence:** 75%
- **provenance:** No GitOps change detected — this is a data/state issue in the S3 WAL archive, not a config change. what_changed(security) and what_changed(flux-system/zitadel) returned 'no changes found'.

## Symptom

Kustomization/zitadel HealthCheckFailed — CNPG cluster recovery blocked by non-empty S3 WAL archive (barman "Expected empty archive")

Affected resource: HelmRelease security/zitadel

## Investigate

- HelmRelease security/zitadel status: Ready=False (InstallFailed) — 'Helm install failed for release security/zitadel with chart zitadel@10.0.4: failed pre-install: failed early due to stalled resources: [Job/security/zitadel-init status: Failed]'
- pod_status (security): 6 pods `xplane-zitadel-cnpg-cluster-1-full-recovery-*` all Failed, exit 1, age 1h, all on node ip-10-0-44-217
- query_logs (plugin-barman-cloud container): 'Starting barman cloud instance plugin' → 'Checking backup destination with barman-cloud-wal-archive' → 'barman-cloud-check-wal-archive checking the first wal' → ERROR: 'WAL archive check failed for server xplane-zitadel-cnpg-cluster: Expected empty archive' → 'Error invoking barman-cloud-check-wal-archive' → 'Error while handling GRPC request'
- query_logs (full-recovery container): 'Restore through plugin detected, proceeding...' → 'Error while restoring a backup' → 'restore error' (x6, 18:45:59Z→18:58:46Z)
- kube_events (security): SQLInstance/xplane-zitadel ComposeResources (x52): 'Composed resource xplane-zitadel-cnpg-cluster is not yet ready' — repeating from 19:04Z to 20:04Z
- kube_events: Backup/xplane-zitadel-cnpg-daily-backup FindingPod (x122): 'Couldn't find target pod xplane-zitadel-cnpg-cluster-1' — no CNPG instance pod exists
- pod_status: no CNPG instance pods (xplane-zitadel-cnpg-cluster-1-1, etc.) exist — only the 6 failed recovery jobs
- HelmRelease security/zitadel status: Ready=False (InstallFailed) — 'failed pre-install: failed early due to stalled resources: [Job/security/zitadel-init status: Failed]'
- Kustomization flux-system/zitadel: Ready=False (HealthCheckFailed) — 'health check failed: failed early due to stalled resources: [HelmRelease/security/zitadel status: Failed]'
- controller_logs (helm-controller, 120m): no zitadel-specific lines — the HelmRelease install hasn't been re-attempted recently, consistent with exhausted retries or the stalled pre-install job blocking
- pod_status: no pods matching job-name=zitadel-init — the failed init job's pods have been cleaned up
- kb_search runbook 'helmrelease-terminal-failed-exhausted-retries': once retries are exhausted, Flux stops retrying and `flux reconcile helmrelease --reset` is required to clear failure counters

## Cause

1. **The CNPG cluster `xplane-zitadel-cnpg-cluster-1` cannot start because its backup recovery fails: the `barman-cloud-check-wal-archive` tool checks the S3 WAL archive destination and finds it is NOT empty (it contains pre-existing WAL files, likely from a previous cluster instance with the same name), producing the error `WAL archive check failed for server xplane-zitadel-cnpg-cluster: Expected empty archive`. This aborts the recovery job (6 failed pods, all exit 1), the CNPG cluster never reaches Ready, and Crossplane's SQLInstance/xplane-zitadel keeps reporting 'Composed resource xplane-zitadel-cnpg-cluster is not yet ready'.** (75%) — change: No GitOps change detected — this is a data/state issue in the S3 WAL archive, not a config change. what_changed(security) and what_changed(flux-system/zitadel) returned 'no changes found'.
2. **The Flux HelmRelease security/zitadel is stuck in InstallFailed state because its pre-install Job (zitadel-init) failed — the init job cannot connect to the CNPG database cluster which never became Ready (root cause #1). The HelmRelease will not retry on its own if install remediation retries are exhausted; a `flux reconcile helmrelease --reset` may be needed after the database is fixed.** (40%)

## Resolution

- Clear the stale WAL archive from the S3 backup destination for the xplane-zitadel-cnpg-cluster server so barman finds an empty archive as expected. The S3 path can be determined from the CNPG Cluster spec's backup configuration (barmanObjectStore.destinationPath). Either delete the WAL files under that prefix (if they belong to a previous/deleted cluster) or change the cluster to restore from the correct backup. Once the S3 archive is clean, CNPG should successfully complete recovery, the cluster will become Ready, and the zitadel HelmRelease pre-install Job will be able to connect to the database. Then reconcile the HelmRelease: `flux -n security reconcile helmrelease zitadel --force` (or `--reset` if install retries are exhausted). (reversible=true)
- After fixing the CNPG cluster (root cause #1), reconcile the HelmRelease to trigger a fresh install: `flux -n security reconcile helmrelease zitadel --force`. If that fails, escalate to `flux -n security reconcile helmrelease zitadel --reset` to clear exhausted install failure counters. If helm release storage is wedged, `helm -n security uninstall zitadel` first, then the --reset reconcile. (reversible=true)

## Unresolved

- Why does the S3 WAL archive destination contain pre-existing WAL files? This could be from a previous CNPG cluster with the same name, a Crossplane SQLInstance that was deleted and recreated (S3 data persists), or a misconfigured destinationPath pointing to a shared bucket. A human with access to the CNPG Cluster spec and S3 bucket must determine the correct fix: either clear the stale WAL archive, change the destinationPath to a unique prefix, or configure the cluster to properly restore from an existing backup instead of expecting an empty archive.
- Whether the Crossplane SQLInstance/xplane-zitadel was recently deleted and recreated (which would explain why the S3 backup data persists while the cluster expects a fresh bootstrap). The Crossplane events only show 'ComposeResources: not yet ready' from 19:04 onward — earlier lifecycle events may be outside the 180m window.

## Citations

[1] No GitOps change detected — this is a data/state issue in the S3 WAL archive, not a config change. what_changed(security) and what_changed(flux-system/zitadel) returned 'no changes found'.

