---
type: Incident
title: Flux Kustomization flux-operator ReconciliationFailed — transient external-secrets webhook outage during GKE node…
description: GKE node gke-gcp-0-static-17bc7341-fvm4 was deleted by GKE's container-engine-robot during a cluster maintenance/upgrade (cluster status=RECONCILING), causing the external-secrets-webhook pod running on it to be evicted. The replacement pod could not schedule due to insufficient CPU (0/3 nodes available), leaving the external-secrets-webhook Service with no endpoints for ~1.5 minutes. During this window, the Flux Kustomization flux-operator's dry-run of ExternalSecret/flux-system/flux-ui-oidc called the validating webhook and received 'no endpoints available for service external-secrets-webhook', causing ReconciliationFailed. The cluster autoscaler added a new node and the webhook pod started at 06:26:31Z; the Kustomization succeeded on its next reconcile at 06:27:59Z. The incident is self-healed — the Kustomization is currently Ready=True.
resource: flux-system/flux-operator
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-09-13T06:42:08Z"
fingerprint: 2ebe6ce647492d64472d94fec60c182c66c58eb472781cd686ebd0f3a2c512c5
confidence: 0.82
provenance:
    - GKE node pool maintenance (container-engine-robot deleted node fvm4 at 06:23:55-06:24:51Z)
---

## Decision

- **why keep:** GKE node gke-gcp-0-static-17bc7341-fvm4 was deleted by GKE's container-engine-robot during a cluster maintenance/upgrade (cluster status=RECONCILING), causing the external-secrets-webhook pod running on it to be evicted. The replacement pod could not schedule due to insufficient CPU (0/3 nodes available), leaving the external-secrets-webhook Service with no endpoints for ~1.5 minutes. During this window, the Flux Kustomization flux-operator's dry-run of ExternalSecret/flux-system/flux-ui-oidc called the validating webhook and received 'no endpoints available for service external-secrets-webhook', causing ReconciliationFailed. The cluster autoscaler added a new node and the webhook pod started at 06:26:31Z; the Kustomization succeeded on its next reconcile at 06:27:59Z. The incident is self-healed — the Kustomization is currently Ready=True.
- **confidence:** 82%
- **provenance:** GKE node pool maintenance (container-engine-robot deleted node fvm4 at 06:23:55-06:24:51Z)

## Symptom

Flux Kustomization flux-operator ReconciliationFailed — transient external-secrets webhook outage during GKE node deletion (self-healed)

Affected resource: Kustomization flux-system/flux-operator

## Investigate

- kube_events (security): Node fvm4 — '06:23:57Z NodeNotReady: Node status is now: NodeNotReady', '06:23:57Z Shutdown: Shutdown manager detected shutdown event', '06:24:55Z DeletingNode: Deleting node because it does not exist in the cloud provider', '06:25:00Z RemovingNode'
- cloud_what_changed: '06:23:55Z v1.compute.instances.delete by service-323586397743@container-engine-robot.iam.gserviceaccount.com' and '06:24:51Z v1.compute.instances.delete' — GKE initiated the node deletion
- kube_events (security): '06:25:08Z Pod/external-secrets-webhook-849686b544-fj57z FailedScheduling (x2): 0/3 nodes are available: 1 node(s) were unschedulable, 2 Insufficient cpu' — replacement pod could not be scheduled
- kube_events (security): '06:25:08Z Pod/external-secrets-webhook-849686b544-q4ts9 Unhealthy (x18): Readiness probe failed: dial tcp 100.65.2.84:8081: connect: connection refused' — old pod going down on deleted node
- gitops_resource_status (Kustomization flux-system/flux-operator): Ready=True, message='Applied revision: latest@sha256:baf553b...'; Events show '06:26:21Z Warning ReconciliationFailed: ExternalSecret/flux-system/flux-ui-oidc dry-run failed ... no endpoints available for service external-secrets-webhook' followed by '06:27:59Z Normal ReconciliationSucceeded' — single transient failure that self-healed
- query_metrics_range kube_node_status_condition: node fvm4 went Ready=true→false at 06:25:00Z; new node gke-gcp-0-static-17bc7341-5tae appeared Ready at 06:27:00Z
- pod_status (security): external-secrets-webhook-849686b544-fj57z Running ready=1/1 age=13m node=gke-gcp-0-static-17bc7341-c37v — webhook pod is healthy now
- cloud_resource_health: GKE cluster gcp-0 status=RECONCILING — maintenance/upgrade in progress explains the node churn
- resource_spec (Deployment security/external-secrets-webhook): spec.replicas=1 — single replica, no fault tolerance
- resource_spec (Deployment security/external-secrets-webhook): pod template labels include app.kubernetes.io/name: external-secrets-webhook
- resource_spec (PDB security/external-secrets-pdb): spec.selector.matchLabels = {app.kubernetes.io/instance: external-secrets, app.kubernetes.io/name: external-secrets} — does NOT match webhook pods which have name=external-secrets-webhook
- resource_spec (PDB security/external-secrets-pdb): status.disruptionsAllowed=0, currentHealthy=1, desiredHealthy=1 — only 1 pod matched (the controller, not the webhook)

## Cause

1. **GKE node gke-gcp-0-static-17bc7341-fvm4 was deleted by GKE's container-engine-robot during a cluster maintenance/upgrade (cluster status=RECONCILING), causing the external-secrets-webhook pod running on it to be evicted. The replacement pod could not schedule due to insufficient CPU (0/3 nodes available), leaving the external-secrets-webhook Service with no endpoints for ~1.5 minutes. During this window, the Flux Kustomization flux-operator's dry-run of ExternalSecret/flux-system/flux-ui-oidc called the validating webhook and received 'no endpoints available for service external-secrets-webhook', causing ReconciliationFailed. The cluster autoscaler added a new node and the webhook pod started at 06:26:31Z; the Kustomization succeeded on its next reconcile at 06:27:59Z. The incident is self-healed — the Kustomization is currently Ready=True.** (82%) — change: GKE node pool maintenance (container-engine-robot deleted node fvm4 at 06:23:55-06:24:51Z)
2. **The external-secrets-webhook Deployment is configured with replicas: 1, making it a single point of failure for all ExternalSecret dry-run validation in the cluster. The existing PodDisruptionBudget 'external-secrets-pdb' has selector app.kubernetes.io/name=external-secrets, which matches the main controller pod but NOT the webhook pods (labeled app.kubernetes.io/name=external-secrets-webhook), so the webhook pods are unprotected by the PDB. Even with PDB protection, node deletion is an involuntary disruption not governed by PDBs, so replicas>1 is the real mitigation.** (45%)

## Resolution

- No immediate action needed — the Kustomization has self-healed and is Ready=True. To prevent recurrence: increase external-secrets-webhook Deployment replicas from 1 to 2+ so a single node loss cannot take the webhook offline, and consider adding pod anti-affinity so replicas land on different nodes. (reversible=true)
- Increase external-secrets-webhook replicas to >=2 and add topology spread constraints or pod anti-affinity. Optionally add a separate PDB for the webhook component (selector app.kubernetes.io/name=external-secrets-webhook) though this alone won't prevent node-loss evictions — replicas>1 is the key fix. (reversible=true)

## Citations

[1] GKE node pool maintenance (container-engine-robot deleted node fvm4 at 06:23:55-06:24:51Z)

