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
- [Harbor Registry Down due to IAM Access Key Quota Limit](harbor-registry-down-due-to-iam-access-key-quota-limit.md) — the Crossplane `AccessKey/xplane-harbor` hit the AWS IAM `AccessKeysPerUser: 2` quota, so the `xplane-harbor-access-key` Secret stays empty and `harbor-registry` fails with `CreateContainerConfigError`.
- [Harbor HelmRelease stuck InstallFailed after a cluster capacity shortage](harbor-helmrelease-terminal-failed.md) — the Harbor install timed out with no schedulable capacity and Flux exhausted its retries; cleared with `helm uninstall` + `flux reconcile --reset`.
- [Application/airflow Degraded — ExternalSecret wrong AWS SM path in dev](application-airflow-degraded.md) — the base `ExternalSecret` referenced a prod-only AWS Secrets Manager key absent in the dev account; fixed with a dev-overlay kustomize patch.
- [RunloreHistoryValidation — synthetic validation alert, no real incident (namespace/workload do not exist)](incidents/runlorehistoryvalidation-synthetic-validation-alert-no-real-incident-namespace-workload-do-not-exist-d383a759.md) — This is a synthetic validation test alert, not a real incident. The alert's own message states it exists only to validate RunLore's recurrence/prior-knowledge notification feature end-to-end, and that the namespace runlore-validation and workload history-check do not exist on the cluster. Independent cluster evidence fully confirms this.
- [vmalert ServiceDown — node 10.0.9.21 CPU saturation (37 pods on 4 cores) causes probe timeouts; Karpenter cannot…](incidents/vmalert-servicedown-node-10-0-9-21-cpu-saturation-37-pods-on-4-cores-causes-probe-timeouts-karpenter-cannot-d2081923.md) — Node ip-10-0-9-21.eu-west-3.compute.internal is CPU-saturated (load1=39.89 spiking to 54.51 on a 4-core node; CPU utilization 99.6-99.7%), causing ALL pods on that node — including vmalert — to fail health probes with 'context deadline exceeded'. vmalert's liveness probe failed 91+ times, triggering container restarts (5→7 restarts, last at 19:05:24Z) and the ServiceDown alert. The node carries 37 pods (highest density in cluster) including CPU-heavy workloads from other namespaces: kyverno-admission-controller (780-1537m, up to 1.5 cores), crossplane (238-493m), provider-aws-s3 (142-320m), vmsingle (176-205m), cilium (162-190m).
