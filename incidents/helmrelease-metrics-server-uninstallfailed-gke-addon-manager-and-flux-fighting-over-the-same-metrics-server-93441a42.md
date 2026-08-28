---
type: Incident
title: HelmRelease/metrics-server UninstallFailed — GKE addon manager and Flux fighting over the same metrics-server…
description: 'A Flux HelmRelease for metrics-server (chart 3.14.0) was installed into kube-system, which already hosts a GKE-managed metrics-server addon (Deployment metrics-server-v1.35.1, GKE image v0.8.0-gke.23). Both manage the same-named Kubernetes resources (Service/metrics-server, ClusterRole/system:metrics-server, ServiceAccount/metrics-server), creating a persistent conflict. GKE''s addon manager keeps overwriting these resources (DriftDetected x105, DriftCorrected x54), and when Flux attempted to uninstall the Helm release, the Service deletion timed out because the GKE addon manager kept recreating it — producing the ''Service/kube-system/metrics-server termination timeout: context deadline exceeded'' error.'
resource: kube-system/metrics-server
tags:
    - runlore
    - incident
    - helmrelease
    - kube-system
timestamp: "2026-08-28T17:28:57Z"
fingerprint: 93441a42edae501a880cd42d3e00c1ef26bf6ca792631e022139724c59fe5330
confidence: 0.82
provenance:
    - HelmRelease metrics-server chart 3.14.0 installed into kube-system where GKE addon metrics-server already exists
---

## Decision

- **why keep:** A Flux HelmRelease for metrics-server (chart 3.14.0) was installed into kube-system, which already hosts a GKE-managed metrics-server addon (Deployment metrics-server-v1.35.1, GKE image v0.8.0-gke.23). Both manage the same-named Kubernetes resources (Service/metrics-server, ClusterRole/system:metrics-server, ServiceAccount/metrics-server), creating a persistent conflict. GKE's addon manager keeps overwriting these resources (DriftDetected x105, DriftCorrected x54), and when Flux attempted to uninstall the Helm release, the Service deletion timed out because the GKE addon manager kept recreating it — producing the 'Service/kube-system/metrics-server termination timeout: context deadline exceeded' error.
- **confidence:** 82%
- **provenance:** HelmRelease metrics-server chart 3.14.0 installed into kube-system where GKE addon metrics-server already exists

## Symptom

HelmRelease/metrics-server UninstallFailed — GKE addon manager and Flux fighting over the same metrics-server resources in kube-system

Affected resource: HelmRelease kube-system/metrics-server

## Investigate

- gitops_resource_status: HelmRelease status shows 'UninstallFailed' with DriftDetected (x105) on 'ClusterRole/system:metrics-server changed' and 'Service/kube-system/metrics-server changed', plus DriftCorrected (x54) — a persistent tug-of-war between two controllers
- resource_spec on Deployment/metrics-server-v1.35.1: annotations 'components.gke.io/component-name: metrics-server' and 'components.gke.io/component-version: 1.35.1-gke.5', image 'europe-west4-artifactregistry.gcr.io/gke-release/gke-release/metrics-server:v0.8.0-gke.23' — this is a GKE-managed addon, not the Helm chart's Deployment
- The HelmRelease inventory (from resource_spec) includes 'kube-system_metrics-server__Service' and '_system__metrics-server_rbac.authorization.k8s.io_ClusterRole' — the exact resources GKE also manages
- resource_spec on Service/metrics-server: live selector is 'k8s-app: metrics-server' (GKE's selector), while the HelmRelease values specify service labels 'kubernetes.io/name: Metrics-server' — the Service was overwritten by GKE, not Helm
- Helm uninstall logs (from gitops_resource_status): 'starting delete resource: Service/metrics-server' at 17:17:00, then 'purge requested' only at 17:22:00 — a 5-minute gap matching the 5m0s context deadline timeout
- controller_logs (helm-controller): 'failed to wait for object to sync in-cache after patching ... context deadline exceeded' and 'Reconciler error ... Service/kube-system/metrics-server termination timeout: context deadline exceeded' at 17:22:10
- workload_ownership on pod metrics-server-v1.35.1-5cb66fbbcc-zg8n8: 'no GitOps tracking label on the top controller — owner unresolved', confirming the Deployment is GKE-managed, not Flux-managed

## Cause

1. **A Flux HelmRelease for metrics-server (chart 3.14.0) was installed into kube-system, which already hosts a GKE-managed metrics-server addon (Deployment metrics-server-v1.35.1, GKE image v0.8.0-gke.23). Both manage the same-named Kubernetes resources (Service/metrics-server, ClusterRole/system:metrics-server, ServiceAccount/metrics-server), creating a persistent conflict. GKE's addon manager keeps overwriting these resources (DriftDetected x105, DriftCorrected x54), and when Flux attempted to uninstall the Helm release, the Service deletion timed out because the GKE addon manager kept recreating it — producing the 'Service/kube-system/metrics-server termination timeout: context deadline exceeded' error.** (82%) — change: HelmRelease metrics-server chart 3.14.0 installed into kube-system where GKE addon metrics-server already exists

## Resolution

- Remove the Flux HelmRelease for metrics-server from kube-system — GKE already provides and manages metrics-server as an addon. The HelmRelease is redundant and its presence causes an ongoing conflict with the GKE addon manager. Suspend and delete the HelmRelease, then let GKE's addon manager fully own the metrics-server resources. The GKE-managed metrics-server pod (currently ready=0/1 with HTTP 500 readiness probe failures) should recover once the conflict stops and the shared ServiceAccount is no longer being deleted by Flux. (reversible=true)

## Unresolved

- Cannot read kube-system ServiceAccount/metrics-server or pod logs due to RBAC restrictions (FORBIDDEN), so cannot directly confirm whether the ServiceAccount was deleted by the Helm uninstall and whether GKE recreated it. The hypothesis that the GKE pod's HTTP 500 is caused by ServiceAccount disruption is inferred from the shared resource inventory, not directly observed.
- Cannot determine when/how the Flux HelmRelease for metrics-server was originally created — what_changed found no GitOps object managing it, and the Kustomization list in flux-system does not show a 'kube-system' Kustomization. The HelmRelease may be orphaned or managed by a Kustomization with a non-obvious name. This needs human investigation to locate and remove the HelmRelease definition from Git.

## Citations

[1] HelmRelease metrics-server chart 3.14.0 installed into kube-system where GKE addon metrics-server already exists

