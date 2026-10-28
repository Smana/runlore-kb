---
type: Incident
title: ImageGalleryNoAvailableReplica — false positive caused by typo in alert rule expression (deployment="image-galery"…
description: 'The alert rule ImageGalleryNoAvailableReplica has a typo in its PromQL expression: it selects deployment="image-galery" (missing the second ''l'') instead of the actual deployment name "image-gallery". Because no Deployment named "image-galery" exists in the apps namespace, kube-state-metrics never emits a kube_deployment_status_replicas_available series with that label. The expression absent(kube_deployment_status_replicas_available{namespace="apps", deployment="image-galery"} > 0) therefore always evaluates to 1 (the absent() condition is always true), so the alert fires permanently regardless of the real deployment''s health. The actual deployment image-gallery is fully healthy: 2/2 replicas available, both pods Running and Ready, status condition Available=True since 2026-10-28T07:22:55Z.'
resource: observability/ImageGalleryNoAvailableReplica
tags:
    - runlore
    - incident
    - alertrule
    - observability
timestamp: "2026-10-28T22:04:28Z"
fingerprint: 3ed20167a43374ae662bca18bd63c0cca82fee3f0da61c68164658163214045c
confidence: 0.93
provenance:
    - Alert rule file /etc/vmalert/rules-out/rules-src-0/rules.yaml, group image-gallery, alert ImageGalleryNoAvailableReplica — typo deployment="image-galery" vs actual deployment name image-gallery
---

## Decision

- **why keep:** The alert rule ImageGalleryNoAvailableReplica has a typo in its PromQL expression: it selects deployment="image-galery" (missing the second 'l') instead of the actual deployment name "image-gallery". Because no Deployment named "image-galery" exists in the apps namespace, kube-state-metrics never emits a kube_deployment_status_replicas_available series with that label. The expression absent(kube_deployment_status_replicas_available{namespace="apps", deployment="image-galery"} > 0) therefore always evaluates to 1 (the absent() condition is always true), so the alert fires permanently regardless of the real deployment's health. The actual deployment image-gallery is fully healthy: 2/2 replicas available, both pods Running and Ready, status condition Available=True since 2026-10-28T07:22:55Z.
- **confidence:** 93%
- **provenance:** Alert rule file /etc/vmalert/rules-out/rules-src-0/rules.yaml, group image-gallery, alert ImageGalleryNoAvailableReplica — typo deployment="image-galery" vs actual deployment name image-gallery

## Symptom

ImageGalleryNoAvailableReplica — false positive caused by typo in alert rule expression (deployment="image-galery" missing an "l")

Affected resource: AlertRule observability/ImageGalleryNoAvailableReplica

## Investigate

- alert_rule(ImageGalleryNoAvailableReplica) returned the expression: absent(kube_deployment_status_replicas_available{namespace="apps", deployment="image-galery"} > 0) — note deployment="image-galery" (typo, missing 'l'), defined in /etc/vmalert/rules-out/rules-src-0/rules.yaml under group image-gallery
- query_metrics(kube_deployment_status_replicas_available{namespace="apps", deployment="image-galery"}) returned 'no series matched' — confirming no metric exists with the misspelled deployment label
- query_metrics(kube_deployment_status_replicas_available{namespace="apps", deployment="image-gallery"}) returned value 2 — the real deployment has 2 available replicas
- query_metrics_range(absent(kube_deployment_status_replicas_available{namespace="apps", deployment="image-galery"} > 0)) over 720m returned a constant value of 1 across the entire window — the alert condition is ALWAYS true, so the alert fires permanently
- query_metrics_range(kube_deployment_status_replicas_available{namespace="apps", deployment="image-gallery"}) over 720m returned a constant value of 2 — the real deployment has had 2 available replicas throughout
- pod_status(apps) shows both image-gallery pods Running and Ready: image-gallery-75d966dbd9-28fj2 ready=2/2 and image-gallery-75d966dbd9-4sb58 ready=2/2 (one with 8 restarts but currently healthy)
- resource_spec(Deployment apps/image-gallery) confirms status.availableReplicas=2, status.readyReplicas=2, status.replicas=2, and condition Available=True with reason MinimumReplicasAvailable since 2026-10-28T07:22:55Z
- list_resources(Deployment in apps) lists 5 deployments; the correct name is 'image-gallery' — no deployment named 'image-galery' exists
- kube_events(apps) returned no Warning events in the namespace — no actual pod or deployment issue

## Cause

1. **The alert rule ImageGalleryNoAvailableReplica has a typo in its PromQL expression: it selects deployment="image-galery" (missing the second 'l') instead of the actual deployment name "image-gallery". Because no Deployment named "image-galery" exists in the apps namespace, kube-state-metrics never emits a kube_deployment_status_replicas_available series with that label. The expression absent(kube_deployment_status_replicas_available{namespace="apps", deployment="image-galery"} > 0) therefore always evaluates to 1 (the absent() condition is always true), so the alert fires permanently regardless of the real deployment's health. The actual deployment image-gallery is fully healthy: 2/2 replicas available, both pods Running and Ready, status condition Available=True since 2026-10-28T07:22:55Z.** (93%) — change: Alert rule file /etc/vmalert/rules-out/rules-src-0/rules.yaml, group image-gallery, alert ImageGalleryNoAvailableReplica — typo deployment="image-galery" vs actual deployment name image-gallery

## Resolution

- Fix the typo in the alert rule expression in /etc/vmalert/rules-out/rules-src-0/rules.yaml (sourced from Git, group image-gallery, alert ImageGalleryNoAvailableReplica): change deployment="image-galery" to deployment="image-gallery" to match the actual Deployment name. After the rule file is corrected and vmalert reloads, the alert will stop firing because the real metric returns value 2 (> 0), making absent() evaluate to 0 (false). No workload remediation is needed — the image-gallery deployment is healthy. (reversible=true)

## Unresolved

- The exact Git source file path and line for the alert rule could not be confirmed from the available tooling (the rule is rendered to /etc/vmalert/rules-out/rules-src-0/rules.yaml by vmalert, but the originating Git repository path was not surfaced by what_changed on the observability namespace, which showed only unrelated CRD Kustomization syncs from August and October). A human should locate the rule source in the monitoring/alerting Git repo and fix the typo there.

## Citations

[1] Alert rule file /etc/vmalert/rules-out/rules-src-0/rules.yaml, group image-gallery, alert ImageGalleryNoAvailableReplica — typo deployment="image-galery" vs actual deployment name image-gallery

