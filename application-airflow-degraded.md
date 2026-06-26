---
type: Incident
title: Application/airflow Degraded — ExternalSecret wrong AWS SM path in dev
description: 'ExternalSecret/wet-collab-database-admin-credentials referenced the prod AWS Secrets Manager path (compute-prod/wet-collab/database/admin) which does not exist in the dev account. Dev uses the rds/ key prefix. Fixed by adding a kustomize patch in the dev-0 overlay to override the key.'
resource: argocd/airflow
tags:
    - runlore
    - incident
    - externalsecrets
    - kustomize
    - dev-0
timestamp: "2026-06-26T12:06:25Z"
resolved: "2026-06-26T14:30:00Z"
fingerprint: 1c203f16cd77d6fc915e6affc8f4b1fbd63d7d27de3dd39a1f337e269ec89a94
---

## Decision

- **why keep:** ExternalSecret `wet-collab-database-admin-credentials` referenced the prod AWS SM path; dev account uses a different path prefix. A kustomize overlay patch is the right fix for env-specific SM key divergence.
- **confidence:** 100% (confirmed resolved)

## Symptom

`Application/airflow` Degraded on dev-0 ArgoCD. The Helm release is Synced but app health stays Degraded.

```
Warning ExternalSecret/wet-collab-database-admin-credentials UpdateFailed (x648):
  error processing spec.dataFrom[0].extract, err: Secret does not exist
```

## Root Cause

The base `ExternalSecret` manifest references:

```yaml
spec:
  dataFrom:
    - extract:
        key: compute-prod/wet-collab/database/admin
```

This path is only valid in the **production** AWS account. The dev account
(`921475819851`) uses the `rds/` key prefix instead. The secret
`compute-prod/wet-collab/database/admin` simply does not exist in the dev
account's Secrets Manager, so ESO's `extract` call fails on every refresh cycle.

The missing secret cascades: Airflow cannot obtain its database credentials →
pods fail to start → Helm release health degrades → ArgoCD marks the app Degraded.

Secondary symptoms observed (not root cause):
- `Multi-Attach` error on Redis PVC — pre-existing unrelated node drain issue
- Scheduler pod `FailedScheduling` / CNI errors — side effects of app degradation

## Resolution

Added a kustomize strategic-merge patch in the dev-0 overlay to override the key:

**`infra/kubernetes/applications/data/airflow/dev-0/patches/fix-wetcollab-secret-path.yaml`**

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: wet-collab-database-admin-credentials
  namespace: airflow
spec:
  dataFrom:
    - extract:
        key: rds/wet-collab/admin
```

Patch wired into `infra/kubernetes/applications/data/airflow/dev-0/kustomization.yaml`.

Merged as [Aqemia/engineering#6202](https://github.com/Aqemia/engineering/pull/6202).

After ArgoCD synced, ExternalSecret transitioned to `Ready: True` and airflow
transitioned to `Healthy`.

## Unresolved

None — root cause confirmed and fix verified in production.

## Generalisation

Whenever a base ESO `ExternalSecret` manifest uses a prod-only AWS SM path
(`compute-prod/…`), dev overlays **must** include a kustomize patch to remap to
the dev-account equivalent (`rds/…`). Consider adding a CI check that flags
`compute-prod/` keys appearing in any non-prod overlay's rendered output.
