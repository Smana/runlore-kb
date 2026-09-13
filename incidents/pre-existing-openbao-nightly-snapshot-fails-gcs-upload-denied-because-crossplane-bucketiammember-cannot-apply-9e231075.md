---
type: Incident
title: '[PRE-EXISTING] OpenBao nightly snapshot fails: GCS upload denied because Crossplane BucketIAMMember cannot apply…'
description: 'The Crossplane BucketIAMMember granting roles/storage.objectCreator on the GCS bucket ''ogenki-435905-ogenki-openbao-snapshot'' to the openbao-snapshot workload identity cannot be applied — GCP returns Error 403 Access Denied on storage.setIamPermissions. This is the SAME fault identified 14h47m ago in occurrence #1; it was never fixed. The Crossplane provider continues to retry and continues to be denied as recently as 04:25:38Z today.'
resource: security/xplane-openbao-snapshot-storage-objectcreator-ogenki-435905-ogenki-openbao-snapshot
alert_resource: security/openbao-snapshot
tags:
    - runlore
    - incident
    - bucketiammember
    - security
timestamp: "2026-09-13T04:27:35Z"
fingerprint: 9e231075f2005a31d2ff1654ee60f14945b78305a78b8185ab29c28b20f17f3f
confidence: 0.9
provenance:
    - No GitOps change since prior incident — the fault persists in the GCP IAM layer; the Crossplane BucketIAMMember status has been False since 2026-09-11T15:22:23Z
---

## Decision

- **why keep:** The Crossplane BucketIAMMember granting roles/storage.objectCreator on the GCS bucket 'ogenki-435905-ogenki-openbao-snapshot' to the openbao-snapshot workload identity cannot be applied — GCP returns Error 403 Access Denied on storage.setIamPermissions. This is the SAME fault identified 14h47m ago in occurrence #1; it was never fixed. The Crossplane provider continues to retry and continues to be denied as recently as 04:25:38Z today.
- **confidence:** 90%
- **provenance:** No GitOps change since prior incident — the fault persists in the GCP IAM layer; the Crossplane BucketIAMMember status has been False since 2026-09-11T15:22:23Z

## Symptom

[PRE-EXISTING] OpenBao nightly snapshot fails: GCS upload denied because Crossplane BucketIAMMember cannot apply storage.objectCreator on the snapshot bucket (occurrence #2, unfixed since 14h47m ago)

Affected resource: BucketIAMMember security/xplane-openbao-snapshot-storage-objectcreator-ogenki-435905-ogenki-openbao-snapshot

## Investigate

- resource_spec on BucketIAMMember xplane-openbao-snapshot-storage-objectcreator-ogenki-435905-ogenki-openbao-snapshot: Synced=False (ReconcileError), Ready=False (Creating) — status.conditions message: 'async create failed: failed to create the resource: [{0 Error applying IAM policy for storage bucket "b/ogenki-435905-ogenki-openbao-snapshot": Error setting IAM policy for storage bucket "b/ogenki-435905-ogenki-openbao-snapshot": googleapi: Error 403: Access denied., forbidden  []}]', lastTransitionTime 2026-09-11T15:22:23Z (unchanged for ~37h)
- pod_logs (all 3 job pods): OpenBao JWT auth succeeds, snapshot is taken successfully ('INFO: Requesting a snapshot via https://openbao.security.svc.cluster.local:8200'), then the GCS upload fails: 'ERROR: (gcloud.storage.cp) You do not currently have an active account selected.' — the pod's SA has no GCS credentials because the IAM binding was never applied
- cloud_what_changed filtered to 'ogenki-openbao-snapshot': repeated 'storage.googleapis.com storage.setIamPermissions — FAILED: PERMISSION_DENIED (Access denied.)' at 04:22:38, 04:23:38, 04:24:38, 04:25:38Z — the Crossplane provider is actively retrying and still being denied
- kube_events on the BucketIAMMember: Warning CannotCreateExternalResource (x2212) with the same 403 Access Denied error — 2212 failed reconciliation attempts
- pod_status: all 3 pods of the Job are Failed with exit code 1; Job status: failed=3, BackoffLimitExceeded at 2026-09-13T04:01:51Z
- workload_ownership: Pod → Job → CronJob/openbao-snapshot → Kustomization flux-system/security-openbao-snapshot (flux), drift=none — no out-of-band changes; the GitOps config is in-sync, the failure is purely in the GCP IAM layer

## Cause

1. **The Crossplane BucketIAMMember granting roles/storage.objectCreator on the GCS bucket 'ogenki-435905-ogenki-openbao-snapshot' to the openbao-snapshot workload identity cannot be applied — GCP returns Error 403 Access Denied on storage.setIamPermissions. This is the SAME fault identified 14h47m ago in occurrence #1; it was never fixed. The Crossplane provider continues to retry and continues to be denied as recently as 04:25:38Z today.** (90%) — change: No GitOps change since prior incident — the fault persists in the GCP IAM layer; the Crossplane BucketIAMMember status has been False since 2026-09-11T15:22:23Z

## Resolution

- Grant the Crossplane GCP provider (or the workload identity pool principal) the storage admin or bucket-level IAM permission to set IAM policies on the snapshot bucket. The bucket IAM member spec wants to bind roles/storage.objectCreator to the principal 'iam.googleapis.com/projects/323586397743/locations/global/workloadIdentityPools/ogenki-435905.svc.id.goog/subject/ns/security/sa/openbao-snapshot' — but the Crossplane provider itself is being denied when calling storage.setIamPermissions. Fix at the GCP project IAM level: either (a) grant the Crossplane provider's GCP identity the 'Storage Admin' or 'roles/storage.legacyBucketOwner' role on the bucket/project, or (b) directly apply the IAM binding via gcloud: `gsutil iam ch principal://iam.googleapis.com/projects/323586397743/locations/global/workloadIdentityPools/ogenki-435905.svc.id.goog/subject/ns/security/sa/openbao-snapshot:roles/storage.objectCreator gs://ogenki-435905-ogenki-openbao-snapshot` — then let the Crossplane BucketIAMMember reconcile. Once the IAM binding is in place, the next nightly CronJob run will succeed automatically. (reversible=true)

## Unresolved

- Why exactly GCP denies the Crossplane provider's storage.setIamPermissions call — the 403 Access Denied is on the Crossplane GCP provider's own identity, which lacks sufficient IAM authority to set bucket IAM policy. The specific GCP project-level IAM role binding needed on the Crossplane provider's service account cannot be determined from Kubernetes-side tooling alone — this requires checking the GCP project IAM policy (gcloud projects get-iam-policy) which is outside the available tools' scope.

## Citations

[1] No GitOps change since prior incident — the fault persists in the GCP IAM layer; the Crossplane BucketIAMMember status has been False since 2026-09-11T15:22:23Z

