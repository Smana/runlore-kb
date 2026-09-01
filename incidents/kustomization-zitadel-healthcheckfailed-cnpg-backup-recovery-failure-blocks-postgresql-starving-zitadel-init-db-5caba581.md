---
type: Incident
title: Kustomization/zitadel HealthCheckFailed — CNPG backup recovery failure blocks PostgreSQL, starving zitadel-init DB…
description: 'The CNPG PostgreSQL cluster xplane-zitadel-cnpg-cluster is bootstrapping from a barman-cloud backup recovery (bootstrap.recovery.source: zitadel-20260829-2, ObjectStore xplane-zitadel-cnpg-objectstore-recovery → s3://eu-west-3-ogenki-cnpg-backups), but WAL recovery consistently fails: PostgreSQL FATAL ''could not locate required checkpoint record at 2/42001828'' (invalid checkpoint record). The required WAL segment at checkpoint LSN 2/42001828 is missing/corrupt in the S3 archive. The full-recovery Job has failed 6+ times since 08:09:23Z (pods: xprzz, xzsbs, kthkb, p8tsq, wp9bs, 8s72t, 6j9mg, all Error exit 1). The CNPG Cluster status is phase=''Cluster is unrecoverable and needs manual intervention'', phaseReason=''Instance creation failed for the following jobs: xplane-zitadel-cnpg-cluster-1-full-recovery''. No PostgreSQL instance ever comes online, so the -rw Service (xplane-zitadel-cnpg-cluster-rw) has no healthy endpoints.'
resource: security/xplane-zitadel-cnpg-cluster
alert_resource: flux-system/zitadel
tags:
    - runlore
    - incident
    - cluster
    - security
timestamp: "2026-09-01T08:25:17Z"
fingerprint: 5caba581b3aa9618f3dad04162805845005895a45dd7bc536aa26d6816ec8e37
confidence: 0.85
---

## Decision

- **why keep:** The CNPG PostgreSQL cluster xplane-zitadel-cnpg-cluster is bootstrapping from a barman-cloud backup recovery (bootstrap.recovery.source: zitadel-20260829-2, ObjectStore xplane-zitadel-cnpg-objectstore-recovery → s3://eu-west-3-ogenki-cnpg-backups), but WAL recovery consistently fails: PostgreSQL FATAL 'could not locate required checkpoint record at 2/42001828' (invalid checkpoint record). The required WAL segment at checkpoint LSN 2/42001828 is missing/corrupt in the S3 archive. The full-recovery Job has failed 6+ times since 08:09:23Z (pods: xprzz, xzsbs, kthkb, p8tsq, wp9bs, 8s72t, 6j9mg, all Error exit 1). The CNPG Cluster status is phase='Cluster is unrecoverable and needs manual intervention', phaseReason='Instance creation failed for the following jobs: xplane-zitadel-cnpg-cluster-1-full-recovery'. No PostgreSQL instance ever comes online, so the -rw Service (xplane-zitadel-cnpg-cluster-rw) has no healthy endpoints.
- **confidence:** 85%

## Symptom

Kustomization/zitadel HealthCheckFailed — CNPG backup recovery failure blocks PostgreSQL, starving zitadel-init DB connection

Affected resource: Cluster security/xplane-zitadel-cnpg-cluster

## Investigate

- resource_spec Cluster/xplane-zitadel-cnpg-cluster: status.phase='Cluster is unrecoverable and needs manual intervention', phaseReason='Instance creation failed for the following jobs: xplane-zitadel-cnpg-cluster-1-full-recovery. Check the job logs to investigate the cause of the failure.', condition Ready=False (ClusterIsNotReady) since 2026-09-01T08:08:28Z
- resource_spec Cluster spec: bootstrap.recovery.source=zitadel-20260829-2, externalClusters[0].plugin.name=barman-cloud.cloudnative-pg.io, parameters.barmanObjectName=xplane-zitadel-cnpg-objectstore-recovery, destinationPath=s3://eu-west-3-ogenki-cnpg-backups
- pod_logs full-recovery (xplane-zitadel-cnpg-cluster-1-full-recovery-6j9mg): 'FATAL ... could not locate required checkpoint record at 2/42001828' then 'startup process (PID 40) exited with exit code 1' then 'restore error: while restoring cluster: while activating instance: error starting PostgreSQL instance: exit status 1'
- pod_status security selector cnpg.io/cluster=xplane-zitadel-cnpg-cluster: 7 full-recovery pods all Failed (Error exit 1), ages 35s to 13m — repeated failures since 08:09
- logs_error_summary security: 'restore error' x4 (08:09:30→08:15:56), 'Error while restoring a backup' x4, 'could not start server' x6 across 6 pods (08:09:30→08:21:18)
- query_logs zitadel-init: 'retrying initial database connection in a second: failed to connect to user=postgres database=postgres: 172.20.39.129:5432 (xplane-zitadel-cnpg-cluster-rw): dial error: timeout: context deadline exceeded' (x2, 08:12:51→08:17:22)
- kube_events security: 'Job/zitadel-init DeadlineExceeded: Job was active longer than specified deadline' at 08:13:31Z; 'HelmRelease/zitadel InstallFailed: Helm install failed ... failed pre-install: failed early due to stalled resources: [Job/security/zitadel-init status: Failed]' at 08:13:35Z
- kube_events security: 'HelmRelease/zitadel UninstallFailed ... failed early due to stalled resources: [Job/security/zitadel-cleanup status: Failed]' at 08:14:41Z; 'Job/zitadel-cleanup DeadlineExceeded' at 08:14:37Z
- controller_logs helm-controller: install at 08:08:29 → 'release is in a failed state' at 08:13:35 → uninstall at 08:13:36 → 'release not installed: no release in storage' → reinstall at 08:14:53 — cycling
- gitops_resource_status HelmRelease security/zitadel: Ready=Unknown (Progressing), message='Running install action with timeout of 45m0s'
- gitops_resource_status Kustomization flux-system/zitadel: Ready=Unknown (Progressing), Warning HealthCheckFailed at 08:13:27 'health check failed after 5m0.049082632s: timeout waiting for: [HelmRelease/security/zitadel status: InProgress]'

## Cause

1. **The CNPG PostgreSQL cluster xplane-zitadel-cnpg-cluster is bootstrapping from a barman-cloud backup recovery (bootstrap.recovery.source: zitadel-20260829-2, ObjectStore xplane-zitadel-cnpg-objectstore-recovery → s3://eu-west-3-ogenki-cnpg-backups), but WAL recovery consistently fails: PostgreSQL FATAL 'could not locate required checkpoint record at 2/42001828' (invalid checkpoint record). The required WAL segment at checkpoint LSN 2/42001828 is missing/corrupt in the S3 archive. The full-recovery Job has failed 6+ times since 08:09:23Z (pods: xprzz, xzsbs, kthkb, p8tsq, wp9bs, 8s72t, 6j9mg, all Error exit 1). The CNPG Cluster status is phase='Cluster is unrecoverable and needs manual intervention', phaseReason='Instance creation failed for the following jobs: xplane-zitadel-cnpg-cluster-1-full-recovery'. No PostgreSQL instance ever comes online, so the -rw Service (xplane-zitadel-cnpg-cluster-rw) has no healthy endpoints.** (85%)
2. **The zitadel HelmRelease install (security/zitadel, chart zitadel@10.0.4) is failing because its pre-install hook Job 'zitadel-init' cannot connect to the PostgreSQL database — it dials xplane-zitadel-cnpg-cluster-rw:172.20.39.129:5432 and gets 'timeout: context deadline exceeded'. With the CNPG cluster unrecoverable (root cause #1), the -rw Service has no working endpoint. The zitadel-init Job (activeDeadlineSeconds: 300) hits DeadlineExceeded, causing Helm's pre-install to fail ('failed early due to stalled resources: [Job/security/zitadel-init status: Failed]'). Helm then attempts uninstall remediation, which also fails (zitadel-cleanup Job DeadlineExceeded), and cycles: install→fail→uninstall→reinstall. The parent Kustomization flux-system/zitadel health check times out waiting for the HelmRelease (InProgress), producing the HealthCheckFailed alert.** (78%)

## Resolution

- The barman-cloud backup used for recovery (zitadel-20260829-2 in s3://eu-west-3-ogenki-cnpg-backups) is missing the WAL segment needed at checkpoint LSN 2/42001828, making the cluster unrecoverable from this backup. A human must either: (1) restore from a different/earlier valid backup that has the required WAL segments, (2) verify the WAL archive in S3 is complete and not truncated/missing segments around timeline 5 / segment 000000050000000200000041, or (3) if no valid backup exists, bootstrap a fresh cluster (initdb) instead of recovery — accepting data loss. After fixing the CNPG cluster, reset the zitadel HelmRelease with 'flux -n security reconcile helmrelease zitadel --reset' so Flux performs a fresh install attempt once the DB is reachable. This is a data-integrity decision that requires human judgment — do NOT blindly re-bootstrap. (reversible=false)
- This is a downstream symptom of root cause #1 (CNPG cluster unrecoverable). Once the PostgreSQL cluster is restored/recovered, the zitadel-init Job will be able to connect and the Helm install should proceed. If the HelmRelease enters a terminal-failed state (exceeded maximum retries), reset with: flux -n security reconcile helmrelease zitadel --reset. If helm release storage is wedged (only failed revisions), run 'helm -n security uninstall zitadel' first, then the --reset reconcile. (reversible=true)

## Unresolved

- Whether the barman-cloud backup 'zitadel-20260829-2' in s3://eu-west-3-ogenki-cnpg-backups was always missing the required WAL segment (a bad/incomplete backup) or whether the WAL archive was truncated/deleted after the backup was taken — a human with S3 access must inspect the backup archive to determine if a valid recovery point exists or if a fresh initdb bootstrap (with data loss) is required.
- Why this cluster is bootstrapping from recovery at all — the cluster spec shows bootstrap.recovery from external cluster 'zitadel-20260829-2' (dated Aug 29), suggesting this is a restore/migration of an existing database. Whether the original cluster's WAL archiving was healthy, and whether this recovery was intentional or a reaction to a prior failure, requires human context.

