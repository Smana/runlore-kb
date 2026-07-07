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
- [Kustomization reconciliation failing due to unhealthy cert-manager webhook, caused by Crossplane IAM permission loss](incidents/kustomization-reconciliation-failing-due-to-unhealthy-cert-manager-webhook-caused-by-crossplane-iam-permission-loss-a63d8db4.md) — Crossplane's AWS IAM permissions were revoked, causing a cascading failure of managed infrastructure in the 'security' namespace. This led to multiple critical admission webhooks (including cert-manager's) becoming unhealthy and unresponsive, which in turn blocked the 'zitadel' Kustomization from being applied.
