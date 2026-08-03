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
- [EKS managed control plane — KubeAPIDown/KubeControllerManagerDown/KubeSchedulerDown are false positives](eks-control-plane-down-alerts-false-positive.md)
- [LLMPlatformSemanticRouterDown on an LLM-free cluster — the opt-in LLM platform is suspended](llm-semantic-router-down-platform-suspended.md)
- [Crossplane KMS Alias stuck Synced=False — kms:CreateAlias denied by an unsatisfiable aws:RequestTag condition](crossplane-kms-createalias-requesttag-accessdenied.md)
- [Pod crashloops with AWS "Access Denied" but the pod spec has no AWS_CONTAINER_* env — Pod Identity was never injected](eks-pod-identity-credentials-never-injected.md) — the pod was admitted before its PodIdentityAssociation was usable, so the webhook injected nothing; injection happens at admission, so no restart recovers it. Recreate the pod. Recurs on every cluster rebuild.

## Incidents

_Learned entries land here as the agent investigates novel, human-sharpened incidents._
- [Harbor Registry Down due to IAM Access Key Quota Limit](harbor-registry-down-due-to-iam-access-key-quota-limit.md) — the Crossplane `AccessKey/xplane-harbor` hit the AWS IAM `AccessKeysPerUser: 2` quota, so the `xplane-harbor-access-key` Secret stays empty and `harbor-registry` fails with `CreateContainerConfigError`. Recurs on every cluster rebuild (the IAM user outlives the cluster); carries the verified fix and a pre-teardown key sweep.
- [Harbor HelmRelease stuck InstallFailed after a cluster capacity shortage](harbor-helmrelease-terminal-failed.md) — the Harbor install timed out with no schedulable capacity and Flux exhausted its retries; cleared with `helm uninstall` + `flux reconcile --reset`.
- [Application/airflow Degraded — ExternalSecret wrong AWS SM path in dev](application-airflow-degraded.md) — the base `ExternalSecret` referenced a prod-only AWS Secrets Manager key absent in the dev account; fixed with a dev-overlay kustomize patch.
- [RunloreHistoryValidation — synthetic validation alert, no real incident (namespace/workload do not exist)](incidents/runlorehistoryvalidation-synthetic-validation-alert-no-real-incident-namespace-workload-do-not-exist-d383a759.md) — This is a synthetic validation test alert, not a real incident. The alert's own message states it exists only to validate RunLore's recurrence/prior-knowledge notification feature end-to-end, and that the namespace runlore-validation and workload history-check do not exist on the cluster. Independent cluster evidence fully confirms this.
- [envoy-gateway HelmRelease terminal-failed: chart v1.8.2 RBAC gap blocks InferencePool watch → healthz fails →…](incidents/envoy-gateway-helmrelease-terminal-failed-chart-v1-8-2-rbac-gap-blocks-inferencepool-watch-healthz-fails-c98cf34b.md) — envoy-gateway pod's health/readiness endpoints never come up because controller-runtime's healthz check fails — its cache reflector cannot watch the InferencePool CRD (inference.networking.k8s.io/v1) due to an RBAC denial: 'inferencepools.inference.networking.k8s.io is forbidden: User system:serviceaccount:envoy-gateway-system:envoy-gateway cannot list resource inferencepools in API group inference.networking.k8s.io at the cluster scope'. The failing healthz check blocks the /healthz and /readyz endpoints (connection refused / 500), so the liveness probe kills the pod repeatedly (7 restarts), the Deployment never becomes Ready, and Helm's --wait times out on every install attempt. envoy-gateway v1.8.2 added an InferencePool EventSource/controller (the log shows 'Starting EventSource ... *unstructured.Unstructured[inference.networking.k8s.io/v1 InferencePool]'), but the chart's ClusterRole (envoy-gateway-gateway-helm-envoy-gateway-role) was not updated to grant list/watch on inferencepools — a chart/RBAC gap in v1.8.2.
- [HelmRelease InstallFailed: a node at Cilium ENI IPAM capacity blocks a DaemonSet pod](incidents/helmrelease-installfailed-node-at-cilium-eni-ipam-capacity-blocks-a-daemonset-pod-790c3116.md) — one node hits its Cilium per-node IPAM ceiling, so a DaemonSet pod there never gets an IP (`cilium-cni failed (add): no IPs currently available on the node`); the DaemonSet stays `InProgress`, Helm's `--wait` times out, and the fault surfaces as an `InstallFailed` HelmRelease in a namespace that looks unrelated. Under `ipam: eni` the ceiling is the instance's ENI limit, not a tunable pool — free an IP by evicting a replicated pod, then a plain `flux reconcile --force` clears it.
- [ImageGalleryUnavailable: AWS Secrets Manager secret manually deleted, breaking ExternalSecret → K8s Secret → pod…](incidents/imagegalleryunavailable-aws-secrets-manager-secret-manually-deleted-breaking-externalsecret-k8s-secret-pod-afe1c077.md) — A manual DeleteSecret operation by IAM user 'smana' at 2026-08-03T17:31:26Z deleted the AWS Secrets Manager secret 'cnpg/xplane-image-gallery/roles/image-gallery-app'. This secret is the source for the ExternalSecret 'xplane-image-gallery-cnpg-image-gallery' in the apps namespace, which populates the Kubernetes Secret of the same name that the image-gallery pods mount as DATABASE_URL credentials. With the AWS secret marked for deletion, the ExternalSecret cannot sync (InvalidRequestException on every reconciliation since 17:31:51Z), the K8s Secret is never created, and all pods sit in CreateContainerConfigError with 0 available replicas.
