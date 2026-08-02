---
type: Incident
title: Harbor Registry Down due to IAM Access Key Quota Limit
description: 'The Crossplane resource `AccessKey/xplane-harbor` has hit an AWS IAM quota limit (`AccessKeysPerUser: 2`), preventing it from creating the credentials required by the Harbor registry. The `xplane-harbor-access-key` Secret stays empty, `harbor-registry` fails with `CreateContainerConfigError: couldn''t find key username`, and the `tooling/harbor` HelmRelease times out InstallFailed. Recurs on every cluster rebuild, because the IAM user outlives the cluster.'
resource: tooling/harbor-registry
alert_resource: tooling/harbor
tags:
    - runlore
    - incident
    - harbor
    - tooling
    - crossplane
    - iam
    - aws
    - helmrelease
    - accesskey
timestamp: "2026-06-28T13:07:56Z"
last_validated: "2026-08-02"
fingerprint: 2d6bd8279304b3e17a5d5e35a55fb0c115ffbeabde820af8cdd2494a4141a60b
---

## Decision

- **why keep:** The Crossplane resource `AccessKey/xplane-harbor` has hit an AWS IAM quota limit (`AccessKeysPerUser: 2`), preventing it from creating the credentials required by the Harbor registry. Confirmed and fully remediated on 2026-08-02 — the two open questions this entry carried are now answered.
- **confidence:** 100% (reproduced and fixed end to end)

## Symptom

Harbor Registry Down due to IAM Access Key Quota Limit

`harbor-registry` sits at `1/2` with `CreateContainerConfigError`, and the
`tooling/harbor` HelmRelease reports:

```
Helm install failed for release tooling/harbor with chart harbor@1.18.3:
timeout waiting for: [Deployment/tooling/harbor-registry status: 'InProgress']
```

The `registryctl` container is healthy; only `registry` fails. That asymmetry is
the tell — `registry` is the one consuming the S3 credentials.

## Investigate

- `pod_status` shows the harbor-registry pod failing with `CreateContainerConfigError: couldn't find key username in Secret tooling/xplane-harbor-access-key`
- `kube_events` for the `tooling` namespace shows a persistent Warning on `AccessKey/xplane-harbor`: `LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2`
- The failed Crossplane `AccessKey` resource is responsible for creating the Secret used by the `harbor-registry` pod
- The Secret **exists but has no `data`** — it is created by Crossplane as the AccessKey's connection secret and only populated once the AccessKey reconciles. An empty-but-present Secret is the signature of this failure, not a missing Secret.

Confirm in four commands:

```bash
# 1. The managed resource is stuck — SYNCED=False, READY=False, empty EXTERNAL-NAME
kubectl get accesskey.iam.aws.m.upbound.io -n tooling xplane-harbor

# 2. The exact quota error
kubectl get accesskey.iam.aws.m.upbound.io -n tooling xplane-harbor \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status} [{.reason}]: {.message}{"\n"}{end}'

# 3. The Secret exists but is empty  ->  `null` / no keys
kubectl get secret -n tooling xplane-harbor-access-key -o json | jq -r '.data // {} | keys'

# 4. The quota is genuinely full, and how old each key is
aws iam list-access-keys --user-name xplane-harbor
```

## Cause

1. **The Crossplane resource `AccessKey/xplane-harbor` has hit an AWS IAM quota limit (`AccessKeysPerUser: 2`), preventing it from creating the credentials required by the Harbor registry.** (100%)

   Full chain: `AccessKey` create returns HTTP 409 `LimitExceeded` → the connection
   Secret `tooling/xplane-harbor-access-key` is never populated → the `registry`
   container cannot resolve its `username` env var → `CreateContainerConfigError` →
   the Deployment never goes Ready → Helm `--wait` times out → HelmRelease `InstallFailed`.

2. **Why it keeps coming back:** the IAM user `xplane-harbor` **outlives the cluster**.
   The platform constitution deliberately withholds IAM *delete* permissions from
   Crossplane (no deletion rights for stateful services — IAM, S3, Route53), so
   tearing down the cluster leaves the user and its access key behind. Each rebuild
   mints a fresh key without reclaiming the old one, so the quota fills after two
   rebuilds and **the third rebuild's Harbor never starts.** Observed 2026-08-02 with
   exactly two orphans, dated `2026-07-21` and `2026-07-24` — one per prior rebuild.

   This is why Harbor uses a static IAM user at all: Harbor's S3 registry storage
   cannot consume EKS Pod Identity (upstream `goharbor/harbor#18686` was closed
   unmerged), so the EPI path other workloads use is unavailable here.

## Resolution

Verified end to end on 2026-08-02 (cluster `mycluster-0`, rebuilt the same day).

1. **Identify the IAM user.** It is named after the Crossplane managed resource:
   `AccessKey/xplane-harbor` → IAM user `xplane-harbor`. No console hunting needed —
   read `spec.forProvider.user` on the MR if you want it confirmed.

2. **Decide which keys are safe to delete.** Compare each key's `CreateDate` against
   the age of the *current* cluster. Any key older than the running cluster belongs to
   a destroyed one and is an orphan. A decisive cross-check: if the `AccessKey` MR has
   an **empty `EXTERNAL-NAME`**, this cluster owns *no* key at all, so **every** key
   listed is an orphan and all of them are safe to delete. That was the case here.
   (reversible=false — deleting an access key is permanent; the secret material cannot
   be recovered. It is safe anyway because nothing holds it.)

   ```bash
   aws iam delete-access-key --user-name xplane-harbor --access-key-id <AKIA...>
   ```

3. **Force the managed resource to retry.** Upjet will not re-attempt the failed async
   create promptly on its own:

   ```bash
   kubectl annotate accesskey.iam.aws.m.upbound.io -n tooling xplane-harbor \
     "reconcile.crossplane.io/trigger=$(date +%s)" --overwrite
   ```

   Within ~30s expect `SYNCED=True READY=True` with a populated `EXTERNAL-NAME`, and
   the Secret carrying `username`, `password`, `attribute.secret`,
   `attribute.ses_smtp_password_v4`.

4. **Restart harbor-registry — it will not self-heal.** This is the step most likely to
   be missed. By the time the Secret lands, the Deployment has already tripped
   `ProgressDeadlineExceeded`, and the pod stays in `CreateContainerConfigError`:

   ```bash
   kubectl rollout restart deploy -n tooling harbor-registry
   kubectl rollout status  deploy -n tooling harbor-registry --timeout=180s
   ```

   Confirm the registry actually reached S3 — it should log `listening on [::]:5000`
   with no storage-driver error:

   ```bash
   kubectl logs -n tooling deploy/harbor-registry -c registry --tail=20
   ```

5. **Clear the HelmRelease.** A plain force-reconcile is enough:

   ```bash
   flux reconcile hr harbor -n tooling --force
   ```

   **`--reset` and `helm uninstall` were _not_ required** in this occurrence, despite
   two hours and 125+ failed AccessKey attempts. Try the plain `--force` first and only
   escalate to the terminal-failed procedure (see
   [helmrelease-terminal-failed-exhausted-retries](helmrelease-terminal-failed-exhausted-retries.md))
   if it actually reports exhausted retries — an unnecessary `helm uninstall` on Harbor
   is disruptive.

### Preventing the recurrence

Sweep the orphaned keys **before** tearing a cluster down, or as a step in the rebuild
runbook — the failure is otherwise guaranteed on every third rebuild:

```bash
aws iam list-access-keys --user-name xplane-harbor \
  --query 'AccessKeyMetadata[].AccessKeyId' --output text \
  | xargs -n1 -r aws iam delete-access-key --user-name xplane-harbor --access-key-id
```

Safe to run while no cluster is up. With a cluster running, exclude the key matching
the `AccessKey` MR's `EXTERNAL-NAME`.

## Unresolved

- (none — both prior open questions were answered on 2026-08-02: the IAM user is
  `xplane-harbor`, and orphan keys are identified by `CreateDate` predating the current
  cluster, or by the MR carrying an empty `EXTERNAL-NAME`.)
