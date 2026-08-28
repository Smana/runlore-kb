---
type: Incident
title: CNPG scheduled backup stuck — barman-cloud plugin 403 on GCS bucket (insufficient storage.buckets.get permission)
description: The barman-cloud plugin cannot access the GCS backup bucket ogenki-435905-ogenki-cnpg-backups — it returns HTTP 403 'storage.buckets.get denied' on every WAL archive and backup attempt. This blocks WAL archiving (ContinuousArchiving=False since 08:45Z) and keeps the scheduled backup stuck in phase=started since 09:00Z. The CNPG cluster was freshly bootstrapped at 08:25Z today, and the Crossplane-provisioned GCS infrastructure (Bucket, BucketIAMMember, GCPWorkloadIdentity) was not fully functional when the backup schedule fired at 09:00Z. The GCPWorkloadIdentity grants roles/storage.objectAdmin, which does NOT include storage.buckets.get — the exact permission the barman-cloud-check-wal-archive command requires to verify the bucket before archiving. Crossplane is still actively patching the BucketIAMMember every ~60s (last seen 12:28:30Z), suggesting the IAM binding may also not have propagated to GCS yet.
resource: security/xplane-zitadel-cnpg-daily-backup-20260828090000
alert_resource: security
tags:
    - runlore
    - incident
    - backup
    - security
timestamp: "2026-08-28T12:30:55Z"
fingerprint: 76784561507803fa6cc79f0b85ae78e5d89bde6d9a0c657c3d87e628b4ee4770
confidence: 0.78
provenance:
    - Cluster bootstrapped at 2026-08-28T08:25Z; Crossplane GCS resources (GCPWorkloadIdentity/BucketIAMMember/ObjectStore) being actively reconciled since then
---

## Decision

- **why keep:** The barman-cloud plugin cannot access the GCS backup bucket ogenki-435905-ogenki-cnpg-backups — it returns HTTP 403 'storage.buckets.get denied' on every WAL archive and backup attempt. This blocks WAL archiving (ContinuousArchiving=False since 08:45Z) and keeps the scheduled backup stuck in phase=started since 09:00Z. The CNPG cluster was freshly bootstrapped at 08:25Z today, and the Crossplane-provisioned GCS infrastructure (Bucket, BucketIAMMember, GCPWorkloadIdentity) was not fully functional when the backup schedule fired at 09:00Z. The GCPWorkloadIdentity grants roles/storage.objectAdmin, which does NOT include storage.buckets.get — the exact permission the barman-cloud-check-wal-archive command requires to verify the bucket before archiving. Crossplane is still actively patching the BucketIAMMember every ~60s (last seen 12:28:30Z), suggesting the IAM binding may also not have propagated to GCS yet.
- **confidence:** 78%
- **provenance:** Cluster bootstrapped at 2026-08-28T08:25Z; Crossplane GCS resources (GCPWorkloadIdentity/BucketIAMMember/ObjectStore) being actively reconciled since then

## Symptom

CNPG scheduled backup stuck — barman-cloud plugin 403 on GCS bucket (insufficient storage.buckets.get permission)

Affected resource: Backup security/xplane-zitadel-cnpg-daily-backup-20260828090000

## Investigate

- resource_spec Backup/xplane-zitadel-cnpg-daily-backup-20260828090000: phase=started, reconciliationStartedAt=2026-08-28T09:00:00Z, method=plugin, pluginConfiguration.name=barman-cloud.cloudnative-pg.io
- resource_spec Cluster/xplane-zitadel-cnpg-cluster: condition ContinuousArchiving status=False reason=ContinuousArchivingFailing message='rpc error: code = Unknown desc = unexpected failure invoking barman-cloud-wal-archive: exit status 2' (lastTransitionTime 2026-08-28T08:29:31Z); condition LastBackupSucceeded status=False reason=BackupStarted (lastTransitionTime 09:00:00Z); cluster bootstrapped at 08:25Z today
- query_logs plugin-barman-cloud: 'ERROR: Can\'t connect to cloud provider: 403 GET https://storage.googleapis.com/storage/v1/b/ogenki-435905-ogenki-cnpg-backups?fields=name&prettyPrint=false: Caller does not have storage.buckets.get access to the Google Cloud Storage bucket. Permission storage.buckets.get denied on resource //storage.googleapis.com/projects/_/buckets/ogenki-435905-ogenki-cnpg-backups (or it may not exist).' (x171, first 11:59:21Z → last 12:28:22Z)
- logs_error_summary plugin-barman-cloud: 'Error while handling GRPC request' x654 (first 08:46:40Z → last 12:26:14Z) and 'Error invoking barman-cloud-check-wal-archive' x654 (first 08:46:40Z → last 12:26:14Z)
- logs_error_summary postgres: 'failed to run wal-archive command' x651 (first 08:45:35Z → last 12:24:26Z) and 'Error while calling ArchiveWAL, failing' x651
- resource_spec GCPWorkloadIdentity/xplane-zitadel-cnpg-backup-identity: status Ready=True Synced=True, spec.bucketRoles=[{bucket: ogenki-435905-ogenki-cnpg-backups, role: roles/storage.objectAdmin}] — roles/storage.objectAdmin does NOT grant storage.buckets.get
- cloud_what_changed: BucketIAMMember xplane-zitadel-cnpg-backup-identity-storage-object-admin-ogenki-435905-ogenki-cnpg-backups being patched by crossplane-system:crossplane every ~60s from at least 12:17 through 12:28Z (still actively reconciling)

## Cause

1. **The barman-cloud plugin cannot access the GCS backup bucket ogenki-435905-ogenki-cnpg-backups — it returns HTTP 403 'storage.buckets.get denied' on every WAL archive and backup attempt. This blocks WAL archiving (ContinuousArchiving=False since 08:45Z) and keeps the scheduled backup stuck in phase=started since 09:00Z. The CNPG cluster was freshly bootstrapped at 08:25Z today, and the Crossplane-provisioned GCS infrastructure (Bucket, BucketIAMMember, GCPWorkloadIdentity) was not fully functional when the backup schedule fired at 09:00Z. The GCPWorkloadIdentity grants roles/storage.objectAdmin, which does NOT include storage.buckets.get — the exact permission the barman-cloud-check-wal-archive command requires to verify the bucket before archiving. Crossplane is still actively patching the BucketIAMMember every ~60s (last seen 12:28:30Z), suggesting the IAM binding may also not have propagated to GCS yet.** (78%) — change: Cluster bootstrapped at 2026-08-28T08:25Z; Crossplane GCS resources (GCPWorkloadIdentity/BucketIAMMember/ObjectStore) being actively reconciled since then

## Resolution

- 1. Grant the GCS service account an additional role that includes storage.buckets.get on the bucket — either roles/storage.legacyBucketReader or roles/storage.admin — via the GCPWorkloadIdentity/Crossplane BucketIAMMember spec. roles/storage.objectAdmin alone is insufficient because barman-cloud-check-wal-archive calls storage.buckets.get to verify the bucket. 2. Alternatively, verify the bucket ogenki-435905-ogenki-cnpg-backups actually exists in GCP (the 403 could also mean the bucket was never created by Crossplane — the Bucket resource status is RBAC-blocked). 3. Once permissions/bucket are fixed, delete the stuck Backup object (kubectl delete backup -n security xplane-zitadel-cnpg-daily-backup-20260828090000) to unblock subsequent scheduled runs, then let the next scheduled backup (or a manual kubectl cnpg backup) create a fresh Backup. 4. Also check the xplane-harbor-cnpg-cluster in the tooling namespace — it has the identical barman-cloud 403 failure on the same bucket. (reversible=true)

## Unresolved

- Whether the GCS bucket ogenki-435905-ogenki-cnpg-backups actually exists — the 403 'storage.buckets.get denied (or it may not exist)' is ambiguous between a missing bucket and a missing IAM permission. The Bucket Crossplane resource status is RBAC-blocked, so a human with GCP console access should verify the bucket exists.
- Whether roles/storage.objectAdmin is intentionally the only role (and barman-cloud's bucket.get check is expected to be covered by another mechanism), or whether the Crossplane composition should also grant a bucket-level reader role — this depends on the Crossplane composition template (xgcpworkloadidentities.cloud.ogenki.io) which I could not inspect.

## Citations

[1] Cluster bootstrapped at 2026-08-28T08:25Z; Crossplane GCS resources (GCPWorkloadIdentity/BucketIAMMember/ObjectStore) being actively reconciled since then

