---
type: Incident
title: 'ImageGalleryUnavailable — fresh namespace bootstrap: pods started before S3 PodIdentityAssociation was ready,…'
description: 'Fresh namespace bootstrap with dependency ordering failure: image-gallery pods were created before the S3 EKS PodIdentityAssociation was ready, so the AWS SDK fell back to EC2 instance metadata (169.254.169.254) for S3 credentials, which times out. Pod gc2vw is stuck in a CrashLoopBackOff (8 restarts) because container restarts do not re-inject EPI env vars — only new pod creation does. Pod ft8lz recovered (ready=2/2) because its restart happened after the EPI agent was ready.'
resource: apps/image-gallery
tags:
    - runlore
    - incident
    - deployment
    - apps
timestamp: "2026-09-16T07:52:21Z"
fingerprint: c7e9ae9259872bc104f0b9d0c9eb3feea886b4f9b156acf62bf5a88f991a52f8
confidence: 0.78
provenance:
    - Fresh namespace bootstrap at ~07:39Z (no GitOps change triggered this — the only GitOps diff is a 3-day-old atlas.sum migration checksum change at 2026-09-13T07:23:27Z)
---

## Decision

- **why keep:** Fresh namespace bootstrap with dependency ordering failure: image-gallery pods were created before the S3 EKS PodIdentityAssociation was ready, so the AWS SDK fell back to EC2 instance metadata (169.254.169.254) for S3 credentials, which times out. Pod gc2vw is stuck in a CrashLoopBackOff (8 restarts) because container restarts do not re-inject EPI env vars — only new pod creation does. Pod ft8lz recovered (ready=2/2) because its restart happened after the EPI agent was ready.
- **confidence:** 78%
- **provenance:** Fresh namespace bootstrap at ~07:39Z (no GitOps change triggered this — the only GitOps diff is a 3-day-old atlas.sum migration checksum change at 2026-09-13T07:23:27Z)

## Symptom

ImageGalleryUnavailable — fresh namespace bootstrap: pods started before S3 PodIdentityAssociation was ready, causing S3 credential IMDS fallback crash-loop

Affected resource: Deployment apps/image-gallery

## Investigate

- pod_logs (previous=true): Pod gc2vw crashes with: 'error: object storage (s3): s3 bucket eu-west-3-ogenki-image-gallery: Get "http://169.254.169.254/latest/meta-data/iam/security-credentials/": dial tcp 169.254.169.254:80: i/o timeout' — the AWS SDK is falling back to EC2 IMDS instead of using EKS Pod Identity
- kube_events: PodIdentityAssociation/xplane-image-gallery-s3-pod-identity-pod-identity-association CannotResolveResourceReferences at 07:39:20Z: 'cannot resolve references: mg.Spec.ForProvider.RoleArn: referenced field was empty (referenced resource may not yet be ready)' — the EPI was NOT ready when pods were being created
- pod_logs: Pod ft8lz (recovered, ready=2/2) logged 'bootstrap complete' and 'web role listening' at 07:42:48Z, while gc2vw (still crashing, 8 restarts) logs the S3/IMDS error at 07:44:00Z and 07:45:41Z on every restart
- pod_status: gc2vw Running ready=0/2 restarts=8 last-exit=07:44:42Z; ft8lz Running ready=2/2 restarts=2 — asymmetric recovery confirms a timing/propagation issue, not a persistent config problem
- kube_events: This is a fresh namespace bootstrap — ServiceAccounts 'image-gallery', 'app-wizard', 'podinfo' were not found at 07:39:18-19Z; Crossplane resources (BucketVersioning, RolePolicyAttachment, PodIdentityAssociation) all had CannotResolveResourceReferences at 07:39:20Z
- resource_spec Cluster/xplane-image-gallery-cnpg-cluster: Ready=True at 07:44:06Z, phase 'Cluster in healthy state', readyInstances=2, currentPrimary=xplane-image-gallery-cnpg-cluster-1
- resource_spec ExternalSecret/xplane-image-gallery-cnpg-image-gallery: Ready=True, 'secret synced' at 07:39:21Z — the AWS Secrets Manager secret is intact (NOT the deleted-secret incident from 2026-08-03)
- kube_events: AtlasMigration TransientErr at 07:39:20Z: 'Secret xplane-image-gallery-cnpg-image-gallery not found' and at 07:41:34Z: 'postgres: querying system variables: dial tcp 172.20.30.149:5432: connect: connection refused' — DB was unreachable during bootstrap
- kube_events: Pods gc2vw and ft8lz had Unhealthy probes (connection refused on :8080 and :8081) from 07:41:43Z through 07:42:08Z — app not listening because dependencies weren't ready

## Cause

1. **Fresh namespace bootstrap with dependency ordering failure: image-gallery pods were created before the S3 EKS PodIdentityAssociation was ready, so the AWS SDK fell back to EC2 instance metadata (169.254.169.254) for S3 credentials, which times out. Pod gc2vw is stuck in a CrashLoopBackOff (8 restarts) because container restarts do not re-inject EPI env vars — only new pod creation does. Pod ft8lz recovered (ready=2/2) because its restart happened after the EPI agent was ready.** (78%) — change: Fresh namespace bootstrap at ~07:39Z (no GitOps change triggered this — the only GitOps diff is a 3-day-old atlas.sum migration checksum change at 2026-09-13T07:23:27Z)
2. **CNPG database cluster took ~5 minutes to bootstrap (07:39:21Z→07:44:06Z Ready), during which image-gallery pods could not serve traffic (readiness/liveness probes failing with 'connection refused'). This was a transient contributing cause that has now resolved — the DB is Ready and the DATABASE_URL ExternalSecret synced successfully.** (35%)

## Resolution

- Delete pod image-gallery-5f9d5f6b5f-gc2vw to force the ReplicaSet to create a new pod that will be processed by the now-ready EPI mutating webhook, injecting the AWS_CONTAINER_CREDENTIALS_FULL_URI env var so the AWS SDK uses EKS Pod Identity instead of falling back to IMDS. Command: kubectl delete pod -n apps image-gallery-5f9d5f6b5f-gc2vw. The Deployment controller will immediately create a replacement pod. (reversible=true)
- No action needed for the DB — it is now Ready. The DATABASE_URL secret is synced and the app can connect. This was a transient bootstrap delay. (reversible=true)

## Unresolved

- Why pod ft8lz recovered but gc2vw did not, despite both being created at ~the same time on the same node with the same ServiceAccount — likely the EPI webhook processed ft8lz after the association was ready but gc2vw before, but I could not read ServiceAccount annotations (FORBIDDEN) to confirm EPI env var injection status on each pod.
- Whether gc2vw will self-heal within the Deployment's 600s progress deadline (progressDeadlineSeconds=600) or requires manual pod deletion — the pod has been crash-looping for ~5 minutes of the 10-minute window.

## Citations

[1] Fresh namespace bootstrap at ~07:39Z (no GitOps change triggered this — the only GitOps diff is a 3-day-old atlas.sum migration checksum change at 2026-09-13T07:23:27Z)

