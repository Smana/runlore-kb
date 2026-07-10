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
- [Harbor registry pod stuck in CreateContainerConfigError — Crossplane AccessKey/xplane-harbor blocked by IAM…](incidents/harbor-registry-pod-stuck-in-createcontainerconfigerror-crossplane-accesskey-xplane-harbor-blocked-by-iam-9fc230d3.md) — The Crossplane resource AccessKey/xplane-harbor cannot create an IAM access key because the target IAM user has already reached the AWS quota limit AccessKeysPerUser: 2. Crossplane's CreateAccessKey call is rejected with LimitExceeded (repeated x582 times), so the Kubernetes Secret tooling/xplane-harbor-access-key is never populated with the required 'username' key. The harbor-registry Deployment references this Secret, so the kubelet refuses to create the registry container with CreateContainerConfigError, leaving the pod Pending. Because the registry Deployment never becomes Ready, the Harbor HelmRelease install timed out and is now InstallFailed. This is a persistent condition (not a fresh change) — the container has been waiting >1h per the alert annotation.
