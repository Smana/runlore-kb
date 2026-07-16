---
type: Playbook
title: Crossplane KMS Alias stuck Synced=False — kms:CreateAlias denied by an aws:RequestTag condition it cannot satisfy
description: A Crossplane KMS Alias managed resource never reconciles (AccessDenied on kms:CreateAlias) even though the provider IAM policy lists that action. Root cause is an aws:RequestTag condition on a statement that includes kms:CreateAlias — an action that does not support request tags — so the permission never applies.
resource: alias.kms.aws.m.upbound.io://*
tags: [crossplane, iam, kms, createalias, accessdenied, request-tag, provider-aws, managed-resource, opentofu, openbao-snapshot]
timestamp: 2026-07-16T00:00:00Z
---

# Symptom

- A Crossplane managed resource such as `alias.kms.aws.m.upbound.io/xplane-openbao-snapshot` stays
  `SYNCED=False` / `READY=False` (Ready reason `Creating`).
- Conditions carry an explicit AWS denial:
  ```
  AccessDeniedException: User: arn:aws:sts::<acct>:assumed-role/<cluster>-crossplane-… is not
  authorized to perform: kms:CreateAlias on resource: arn:aws:kms:<region>:<acct>:alias/xplane-…
  because no identity-based policy allows the kms:CreateAlias action
  ```

# Investigate

1. Read the MR conditions for the exact denied action + resource ARN:
   ```
   kubectl get alias.kms.aws.m.upbound.io -n <ns> <name> \
     -o jsonpath='{range .status.conditions[*]}{.type}={.status} {.reason}: {.message}{"\n"}{end}'
   ```
2. Check the Crossplane provider IAM policy (here: `opentofu/eks/init/iam.tf`,
   `aws_iam_policy.crossplane_kms`). The trap: the policy **does** list `kms:CreateAlias`, so a
   naive "add the action" reading says it should work. Look at the **Condition** on that statement:
   ```json
   { "Action": ["kms:CreateKey","kms:TagResource","kms:CreateAlias"],
     "Resource": ["*"],
     "Condition": { "StringLike": { "aws:RequestTag/crossplane-name": "xplane-*" } } }
   ```

# Root cause

`kms:CreateAlias` **does not support request tags** — an alias cannot be tagged on creation, so there
is no `aws:RequestTag/*` in the request context. A `StringLike aws:RequestTag/...` condition can
never match for `CreateAlias`, so IAM evaluates the statement as not-applicable → the action is
implicitly denied. The condition is only valid for tag-on-create actions like `kms:CreateKey`.
Symptom is "the action is literally in the policy but still denied."

# Resolution

- Split `kms:CreateAlias` into its own statement scoped by **resource ARN** instead of a request-tag
  condition, keeping tag-conditioned actions (`kms:CreateKey`) as they are:
  ```json
  { "Effect": "Allow",
    "Action": "kms:CreateAlias",
    "Resource": [
      "arn:aws:kms:*:<acct>:alias/xplane-*",
      "arn:aws:kms:*:<acct>:key/*"
    ] }
  ```
  (`CreateAlias` authorizes against **both** the alias and its target key.)
- This is an OpenTofu IAM change — it requires `terramate/tofu apply` on `eks/init`; it is **not**
  fixed through the Flux/GitOps loop. Sequence it as an infra change, then let the MR retry.

# Verification

- After apply, the MR reconciles `SYNCED=True` / `READY=True`.
- `aws kms list-aliases --region <region> | grep xplane-` shows the alias created.

# Citations

- AWS KMS: `CreateAlias` does not support tag-on-create / `aws:RequestTag` condition keys.
- Crossplane provider-aws (`m.upbound.io`) v2 managed resources are namespaced; MR conditions carry
  the raw AWS API error.
