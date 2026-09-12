---
type: Incident
title: Kustomization/security-tailscale HealthCheckCanceled — transient cascade from cert-manager HelmRelease upgrade…
description: A cert-manager HelmRelease upgrade (chart changed) triggered by the security ExternalArtifact update is stuck Progressing because the new cert-manager and cainjector pods can't schedule — GKE node pool capacity was insufficient. The security Kustomization remains Progressing (not failed), which cascades as DependencyNotReady to security-openbao → security-tailscale, producing the HealthCheckCanceled Normal event when Flux cancels a stale health check during re-reconciliation.
resource: flux-system/security-tailscale
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-09-12T22:07:07Z"
fingerprint: e52600f18de12f9f5622e0f319c3eb4a8ff81da5a2a61965672d774f0bbaf601
confidence: 0.78
---

## Decision

- **why keep:** A cert-manager HelmRelease upgrade (chart changed) triggered by the security ExternalArtifact update is stuck Progressing because the new cert-manager and cainjector pods can't schedule — GKE node pool capacity was insufficient. The security Kustomization remains Progressing (not failed), which cascades as DependencyNotReady to security-openbao → security-tailscale, producing the HealthCheckCanceled Normal event when Flux cancels a stale health check during re-reconciliation.
- **confidence:** 78%

## Symptom

Kustomization/security-tailscale HealthCheckCanceled — transient cascade from cert-manager HelmRelease upgrade blocked by CPU capacity

Affected resource: Kustomization flux-system/security-tailscale

## Investigate

- gitops_resource_status Kustomization/security-tailscale: Ready=False (DependencyNotReady), message: dependency 'flux-system/security-openbao' is not ready; Events show Normal HealthCheckCanceled 'Health checks canceled due to new reconciliation triggered by ExternalArtifact/flux-system/security-artifact' at 22:01:39Z
- gitops_tree: security-tailscale (Ready=False, DependencyNotReady) → security-openbao (Ready=False, DependencyNotReady) → security (Ready=Unknown, Progressing). The root not-Ready node is security, which is Progressing, not failed.
- gitops_resource_status Kustomization/security: Ready=Unknown (Progressing), message: Reconciliation in progress. Event at 22:01:40Z: Normal Progressing HelmRelease/security/cert-manager configured
- gitops_resource_status HelmRelease/security/cert-manager: Ready=Unknown (Progressing), message: Running 'upgrade' action with timeout of 5m0s
- controller_logs helm-controller: at 22:01:41Z 'release out-of-sync with desired state: release chart changed' → 'running upgrade action with timeout of 5m0s'
- kube_events: Pod/cert-manager-7448f66cd9-bxh7t FailedScheduling (x7) at 22:03:29Z: '0/4 nodes are available: 1 node(s) had untolerated taint(s), 3 Insufficient cpu'; same for cert-manager-cainjector-5564d6bfff-hd6xf
- pod_status security/cert-manager: cert-manager-7448f66cd9-bxh7t Pending ready=0/1 age=2m (new pod from the upgrade); cert-manager-c6b895db5-k2qkq Running ready=1/1 age=1d (old pod still serving)
- kube_events: at 22:04:44Z Warning Pod/cert-manager-7448f66cd9-bxh7t FailedCreatePodSandBox: 'failed to setup network for sandbox ... plugin type="cilium-cni" failed (add): unable to create endpoint: Put "http://localhost/v1/endpoint/cilium-local:0": EOF' — same for cainjector
- kube_events: at 22:04:22Z Normal Pod/cert-manager-7448f66cd9-bxh7t Scheduled: Successfully assigned to gke-gcp-0-nap-e2-standard-4-5gw594hh-6a897f09-5qlv (the newly-added node)
- pod_logs: cert-manager-7448f66cd9-bxh7t/cert-manager-controller is waiting to start: ContainerCreating (past scheduling, waiting on CNI)
- cloud_resource_health: new node gke-gcp-0-nap-e2-standard-4-5gw594hh-6a897f09-5qlv became Ready at 22:04:00Z; cluster grew from 4→5 nodes; no instance-group errors

## Cause

1. **A cert-manager HelmRelease upgrade (chart changed) triggered by the security ExternalArtifact update is stuck Progressing because the new cert-manager and cainjector pods can't schedule — GKE node pool capacity was insufficient. The security Kustomization remains Progressing (not failed), which cascades as DependencyNotReady to security-openbao → security-tailscale, producing the HealthCheckCanceled Normal event when Flux cancels a stale health check during re-reconciliation.** (78%)
2. **After the new node joined and pods were scheduled, Cilium CNI was not yet ready on the freshly-booted node, causing FailedCreatePodSandBox errors (cilium-cni 'unable to create endpoint: EOF'). This is a transient cold-start condition — the Cilium agent on the new node needs seconds to initialize before it can wire pod networking.** (35%)

## Resolution

- Wait for the GKE autoscaler-provisioned node to finish booting and the Cilium CNI to initialize; the cert-manager pods should then start, the HelmRelease upgrade will complete, and the security → security-openbao → security-tailscale dependency chain will self-heal. If the HelmRelease 5m upgrade timeout fires first, manually reconcile: `flux reconcile helmrelease cert-manager -n security` or `flux reconcile kustomization security -n flux-system --with-source`. If the Cilium CNI error persists on the new node, investigate the Cilium agent on that node. (reversible=true)
- No action needed — the Cilium agent self-initializes on new nodes and the kubelet retries sandbox creation. If pods are still ContainerCreating after ~2 minutes, check the Cilium agent (cilium DaemonSet) on the new node. (reversible=true)

