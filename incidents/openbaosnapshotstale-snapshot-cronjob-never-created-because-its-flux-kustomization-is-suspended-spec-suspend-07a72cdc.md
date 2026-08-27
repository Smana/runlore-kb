---
type: Incident
title: OpenBaoSnapshotStale — snapshot CronJob never created because its Flux Kustomization is suspended (spec.suspend:…
description: 'The Flux Kustomization flux-system/security-openbao-snapshot — which manages the openbao-snapshot CronJob — is explicitly suspended (spec.suspend: true) and has never reconciled (status.observedGeneration: -1). Flux''s kustomize-controller logs confirm on every trigger: ''Reconciliation is suspended for this object''. Because the Kustomization never ran, the CronJob manifest at ./security/gcp-0/openbao-snapshot was never applied; resource_spec confirms CronJob security/openbao-snapshot is ABSENT from the API server. With no CronJob in the cluster, kube_cronjob_status_last_successful_time has no series, so the alert''s absent(...) == 1 branch fired — ''the CronJob has never succeeded once''. This is not a transient dependency failure: the parent Kustomization security-openbao is Ready=True, and the source ExternalArtifact/security-artifact is Ready=True.'
resource: flux-system/security-openbao-snapshot
alert_resource: security/openbao-snapshot
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-08-27T15:06:08Z"
fingerprint: 07a72cdc0ecb18f9a78ba0fee7b6042023bc3d266952cb48b22efe2244517f3e
confidence: 0.82
---

## Decision

- **why keep:** The Flux Kustomization flux-system/security-openbao-snapshot — which manages the openbao-snapshot CronJob — is explicitly suspended (spec.suspend: true) and has never reconciled (status.observedGeneration: -1). Flux's kustomize-controller logs confirm on every trigger: 'Reconciliation is suspended for this object'. Because the Kustomization never ran, the CronJob manifest at ./security/gcp-0/openbao-snapshot was never applied; resource_spec confirms CronJob security/openbao-snapshot is ABSENT from the API server. With no CronJob in the cluster, kube_cronjob_status_last_successful_time has no series, so the alert's absent(...) == 1 branch fired — 'the CronJob has never succeeded once'. This is not a transient dependency failure: the parent Kustomization security-openbao is Ready=True, and the source ExternalArtifact/security-artifact is Ready=True.
- **confidence:** 82%

## Symptom

OpenBaoSnapshotStale — snapshot CronJob never created because its Flux Kustomization is suspended (spec.suspend: true, observedGeneration: -1)

Affected resource: Kustomization flux-system/security-openbao-snapshot

## Investigate

- alert_rule OpenBaoSnapshotStale: expr = 'time() - kube_cronjob_status_last_successful_time{...} > 129600 or absent(kube_cronjob_status_last_successful_time{...}) == 1', for=30m, state=firing
- resource_spec Kustomization/security-openbao-snapshot: spec.suspend: true, status.observedGeneration: -1 (never reconciled); spec.path=./security/gcp-0/openbao-snapshot; spec.dependsOn=[security-openbao]
- resource_spec CronJob security/openbao-snapshot: ABSENT — the API server reports no such object
- controller_logs kustomize-controller (resource=security-openbao-snapshot, since 180m): 'Reconciliation is suspended for this object' at 2026-08-27T15:01:15Z
- gitops_resource_status Kustomization/security-openbao-snapshot: Ready=Unknown, dependsOn=security-openbao, sourceRef=ExternalArtifact/security-artifact
- gitops_tree: security-openbao-snapshot (Ready=unknown) → security-openbao (Ready=True) → security (Ready=True) → crds (Ready=True) → namespaces (Ready=True) — full dependency chain is healthy
- query_metrics kube_cronjob_status_last_successful_time{namespace=security,cronjob=openbao-snapshot}: 'no series matched' — consistent with the CronJob being absent
- discover_metrics security, cronjob label: only 'openbao-snapshot' value exists, but only ALERTS and ALERTS_FOR_STATE metric names are present (the alert itself) — no kube_cronjob_status_* metrics, confirming the CronJob was never observed by kube-state-metrics
- gitops_resource_status Kustomization/security-openbao: Ready=True, ReconciliationSucceeded — the parent dependency is healthy and reconciling normally
- gitops_resource_status ExternalArtifact/security-artifact: Ready=True, 'Artifact is ready' — the source is healthy

## Cause

1. **The Flux Kustomization flux-system/security-openbao-snapshot — which manages the openbao-snapshot CronJob — is explicitly suspended (spec.suspend: true) and has never reconciled (status.observedGeneration: -1). Flux's kustomize-controller logs confirm on every trigger: 'Reconciliation is suspended for this object'. Because the Kustomization never ran, the CronJob manifest at ./security/gcp-0/openbao-snapshot was never applied; resource_spec confirms CronJob security/openbao-snapshot is ABSENT from the API server. With no CronJob in the cluster, kube_cronjob_status_last_successful_time has no series, so the alert's absent(...) == 1 branch fired — 'the CronJob has never succeeded once'. This is not a transient dependency failure: the parent Kustomization security-openbao is Ready=True, and the source ExternalArtifact/security-artifact is Ready=True.** (82%)

## Resolution

- Resume the suspended Kustomization so Flux applies the CronJob manifest: flux -n flux-system resume kustomization security-openbao-snapshot (then flux -n flux-system reconcile kustomization security-openbao-snapshot --with-source to trigger immediately). After reconciliation, verify the CronJob is created (kubectl -n security get cronjob openbao-snapshot) and that it runs successfully at the next scheduled time. If a reason existed for the suspension (e.g. the KB-documented Crossplane KMS CreateAlias IAM denial for the xplane-openbao-snapshot KMS key/alias), verify that prerequisite is resolved before resuming — the snapshot job likely needs a KMS-encrypted S3 bucket/key that must exist first. (reversible=true)

## Unresolved

- WHY the Kustomization is suspended: the spec.suspend: true flag is set in Git, but the what_changed tool could not resolve a diffable GitOps object for this Kustomization (it has never reconciled, so it carries no applied revision). I could not determine whether it was intentionally suspended (e.g. pending the KB-documented Crossplane KMS CreateAlias IAM fix for xplane-openbao-snapshot) or left suspended by mistake. A human should confirm the original reason for suspension before resuming.
- Whether the snapshot CronJob's prerequisites are fully in place: the KB runbook documents a Crossplane KMS Alias (xplane-openbao-snapshot) stuck Synced=False due to kms:CreateAlias being denied by an aws:RequestTag condition. The KMS managed-resource kinds (Alias/Key in kms.aws.m.upbound.io) are not served on this cluster (UNKNOWN KIND), so I could not verify whether the KMS key/alias the snapshot bucket depends on actually exists. If the KMS prerequisite is unresolved, resuming the Kustomization may create the CronJob but it could fail on execution.

