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
- [vllm-semantic-router absent — new llm-platform umbrella Kustomization shipped suspended, blocking all LLM child…](incidents/vllm-semantic-router-absent-new-llm-platform-umbrella-kustomization-shipped-suspended-blocking-all-llm-child-180bbc90.md) — A new Flux 'llm-platform' umbrella Kustomization was committed to the cluster's flux-system sync with spec.suspend: true and prune: true. This is an intentional opt-in gate: while suspended, Flux does NOT create any of the child Kustomizations under clusters/mycluster-0-llm-platform/ — explicitly including vllm-semantic-router. With prune: true, Flux actively removes any previously-applied children. The result: the vllm-semantic-router Kustomization, its namespace pods, and all LLM-platform resources are gone from the cluster, so vmagent has no up{} target to scrape → the critical 'no live up samples' alert fires.
