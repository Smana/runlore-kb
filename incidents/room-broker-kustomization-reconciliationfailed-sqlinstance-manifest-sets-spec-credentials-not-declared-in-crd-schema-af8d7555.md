---
type: Incident
title: 'room-broker Kustomization ReconciliationFailed: SQLInstance manifest sets .spec.credentials not declared in CRD schema'
description: The newly introduced room-broker Kustomization (path ./infrastructure/gcp-0/room-broker, revision 4e2848c250ef, firstReconciled 22:41:26Z) renders a SQLInstance/agent-system/xplane-rooms manifest that sets .spec.credentials, but the SQLInstance CRD (sqlinstances.cloud.ogenki.io) OpenAPI v3 schema does NOT declare a credentials property under .spec. Kubernetes structural-schema CRDs reject unknown fields during server-side-apply typed-patch creation, so the dry-run fails before any apply and the Kustomization stays Ready=False, retrying every 30s. The room-broker Deployment and the SQLInstance managed resource cannot be created until this mismatch is resolved.
resource: flux-system/room-broker
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-09-30T22:50:45Z"
fingerprint: af8d7555d1742932f85e0af26ec29e5e42c473b7c9a6cece6c28c7b8c9604a5a
confidence: 0.78
provenance:
    - revision latest@sha256:4e2848c250efe9500a675d597674ffc264709564fbd3f8946402b7fa34adb234 (integration/agent-factory branch); room-broker Kustomization firstReconciled 2026-09-30T22:41:26Z
---

## Decision

- **why keep:** The newly introduced room-broker Kustomization (path ./infrastructure/gcp-0/room-broker, revision 4e2848c250ef, firstReconciled 22:41:26Z) renders a SQLInstance/agent-system/xplane-rooms manifest that sets .spec.credentials, but the SQLInstance CRD (sqlinstances.cloud.ogenki.io) OpenAPI v3 schema does NOT declare a credentials property under .spec. Kubernetes structural-schema CRDs reject unknown fields during server-side-apply typed-patch creation, so the dry-run fails before any apply and the Kustomization stays Ready=False, retrying every 30s. The room-broker Deployment and the SQLInstance managed resource cannot be created until this mismatch is resolved.
- **confidence:** 78%
- **provenance:** revision latest@sha256:4e2848c250efe9500a675d597674ffc264709564fbd3f8946402b7fa34adb234 (integration/agent-factory branch); room-broker Kustomization firstReconciled 2026-09-30T22:41:26Z

## Symptom

room-broker Kustomization ReconciliationFailed: SQLInstance manifest sets .spec.credentials not declared in CRD schema

Affected resource: Kustomization flux-system/room-broker

## Investigate

- gitops_resource_status(Kustomization/flux-system/room-broker): Ready=False (ReconciliationFailed), message: 'SQLInstance/agent-system/xplane-rooms dry-run failed: failed to create typed patch object (agent-system/xplane-rooms; cloud.ogenki.io/v1alpha1, Kind=SQLInstance): .spec.credentials: [REDACTED] not declared in schema'
- resource_spec(CRD sqlinstances.cloud.ogenki.io): .spec schema declares atlasSchema, backup, createSuperuser, crossplane, databases, initSQL, instances, managementPolicies, objectStoreRecovery, performanceInsights — credentials is NOT among them. CRD Established at 18:22Z (well before the incident).
- controller_logs(kustomize-controller, resource=room-broker): error repeats every 30s from 22:41:26Z onward — 'Reconciliation failed ... error: SQLInstance/agent-system/xplane-rooms dry-run failed: failed to create typed patch object ... .spec.credentials: [REDACTED] not declared in schema'
- resource_spec(Kustomization/flux-system/room-broker): firstReconciled 2026-09-30T22:41:26Z (brand-new Kustomization), path ./infrastructure/gcp-0/room-broker, sourceRef ExternalArtifact/flux-system/infra-artifact @ revision 4e2848c250ef
- resource_spec(SQLInstance agent-system/xplane-rooms): ABSENT — the object was never created because dry-run fails before apply
- gitops_tree(Kustomization/flux-system/room-broker): all dependsOn are Ready (agent-secrets=True, agent-policies=True, crossplane-configuration=True) — NOT a dependency cascade
- resource_spec(Kustomization/flux-system/crossplane-configuration): Ready=True at the same revision 4e2848c250ef — the XRD/CRD schema as committed does not include .spec.credentials, confirming the schema and the instance manifest are out of sync within the same revision

## Cause

1. **The newly introduced room-broker Kustomization (path ./infrastructure/gcp-0/room-broker, revision 4e2848c250ef, firstReconciled 22:41:26Z) renders a SQLInstance/agent-system/xplane-rooms manifest that sets .spec.credentials, but the SQLInstance CRD (sqlinstances.cloud.ogenki.io) OpenAPI v3 schema does NOT declare a credentials property under .spec. Kubernetes structural-schema CRDs reject unknown fields during server-side-apply typed-patch creation, so the dry-run fails before any apply and the Kustomization stays Ready=False, retrying every 30s. The room-broker Deployment and the SQLInstance managed resource cannot be created until this mismatch is resolved.** (78%) — change: revision latest@sha256:4e2848c250efe9500a675d597674ffc264709564fbd3f8946402b7fa34adb234 (integration/agent-factory branch); room-broker Kustomization firstReconciled 2026-09-30T22:41:26Z

## Resolution

- Decide whether .spec.credentials should be a valid field: (a) if yes, add it to the SQLInstance XRD schema in ./infrastructure/gcp-0/crossplane/configuration and re-apply crossplane-configuration so the CRD schema is updated; (b) if no, remove .spec.credentials from the SQLInstance manifest in ./infrastructure/gcp-0/room-broker. Then flux reconcile kustomization room-broker --with-source. To stop the 30s retry churn immediately while deciding, flux suspend kustomization room-broker. (reversible=true)

## Unresolved

- Whether .spec.credentials is an intended new field (schema needs updating in the XRD) or an erroneous field in the manifest (needs removal) — a human who knows the room-broker design intent must decide. Both the SQLInstance CRD schema and the room-broker manifests were committed in the same revision 4e2848c250ef, so this is a within-revision coordination error, not a version skew.

## Citations

[1] revision latest@sha256:4e2848c250efe9500a675d597674ffc264709564fbd3f8946402b7fa34adb234 (integration/agent-factory branch); room-broker Kustomization firstReconciled 2026-09-30T22:41:26Z

