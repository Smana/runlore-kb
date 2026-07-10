---
okf_version: "0.1"
type: Index
title: RunLore Knowledge Catalog
description: Seed runbooks and learned incident knowledge for the RunLore SRE agent.
---

# RunLore Knowledge Catalog

This directory is an [OKF](https://github.com/GoogleCloudPlatform/knowledge-catalog) bundle: a tree
of markdown files with YAML frontmatter. RunLore **reads** it (cached + indexed) to ground
investigations, and **writes** new entries to it (via reviewed PR) as it learns.

Reserved files: `index.md` (this listing) and `log.md` (chronological change history). Every other
`.md` is a concept document. Links between entries assert relationships; the relationship kind is
conveyed by the surrounding prose.

## Playbooks

- [HelmRelease upgrade/install failure](helmrelease-upgrade-failure.md)
- [Flux Kustomization reconciliation failure](kustomization-reconciliation-failure.md)
- [Karpenter EC2NodeClass not ready — AMI alias not found](karpenter-ec2nodeclass-ami-not-found.md)
- [HelmRelease stuck terminal-failed — exhausted retries after a transient install timeout](helmrelease-terminal-failed-exhausted-retries.md)
- [Flux bootstrap — Kustomizations DependencyNotReady until ArtifactGenerator artifacts exist](flux-bootstrap-externalartifact-dependency-cascade.md)

## Incidents

_Learned entries land here as the agent investigates novel, human-sharpened incidents._
- [RunloreHistoryValidation — synthetic validation alert, no real incident (namespace/workload do not exist)](incidents/runlorehistoryvalidation-synthetic-validation-alert-no-real-incident-namespace-workload-do-not-exist-d383a759.md) — This is a synthetic validation test alert, not a real incident. The alert's own message states it exists only to validate RunLore's recurrence/prior-knowledge notification feature end-to-end, and that the namespace runlore-validation and workload history-check do not exist on the cluster. Independent cluster evidence fully confirms this.
- [harbor-registry pod down — Crossplane AccessKey blocked by AWS IAM quota (AccessKeysPerUser: 2)](incidents/harbor-registry-pod-down-crossplane-accesskey-blocked-by-aws-iam-quota-accesskeysperuser-2-3ec63515.md) — The Crossplane resource AccessKey/xplane-harbor cannot create the IAM access key Harbor needs because the associated IAM user has already hit the AWS quota of 2 access keys per user (LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2). With the key never created, the Secret tooling/xplane-harbor-access-key is missing its `username` key, so the harbor-registry container fails to start with CreateContainerConfigError and the pod stays Pending.
