---
type: Incident
title: 'ImageGalleryUnavailable: AWS Secrets Manager secret manually deleted, breaking ExternalSecret → K8s Secret → pod…'
description: A manual DeleteSecret operation by IAM user 'smana' at 2026-08-03T17:31:26Z deleted the AWS Secrets Manager secret 'cnpg/xplane-image-gallery/roles/image-gallery-app'. This secret is the source for the ExternalSecret 'xplane-image-gallery-cnpg-image-gallery' in the apps namespace, which populates the Kubernetes Secret of the same name that the image-gallery pods mount as DATABASE_URL credentials. With the AWS secret marked for deletion, the ExternalSecret cannot sync (InvalidRequestException on every reconciliation since 17:31:51Z), the K8s Secret is never created, and all pods sit in CreateContainerConfigError with 0 available replicas.
resource: apps/xplane-image-gallery
tags:
    - runlore
    - incident
    - deployment
    - apps
timestamp: "2026-08-03T18:06:55Z"
fingerprint: afe1c077a15b7f5465235bf509fe860e53c1327f74e9fb175331b973e8cf4cec
confidence: 0.92
provenance:
    - CloudTrail DeleteSecret by smana at 2026-08-03T17:31:26Z on AWS::SecretsManager::Secret/cnpg/xplane-image-gallery/roles/image-gallery-app
---

## Decision

- **why keep:** A manual DeleteSecret operation by IAM user 'smana' at 2026-08-03T17:31:26Z deleted the AWS Secrets Manager secret 'cnpg/xplane-image-gallery/roles/image-gallery-app'. This secret is the source for the ExternalSecret 'xplane-image-gallery-cnpg-image-gallery' in the apps namespace, which populates the Kubernetes Secret of the same name that the image-gallery pods mount as DATABASE_URL credentials. With the AWS secret marked for deletion, the ExternalSecret cannot sync (InvalidRequestException on every reconciliation since 17:31:51Z), the K8s Secret is never created, and all pods sit in CreateContainerConfigError with 0 available replicas.
- **confidence:** 92%
- **provenance:** CloudTrail DeleteSecret by smana at 2026-08-03T17:31:26Z on AWS::SecretsManager::Secret/cnpg/xplane-image-gallery/roles/image-gallery-app

## Symptom

ImageGalleryUnavailable: AWS Secrets Manager secret manually deleted, breaking ExternalSecret → K8s Secret → pod startup

Affected resource: Deployment apps/xplane-image-gallery

## Investigate

- CloudTrail: '2026-08-03T17:31:26Z secretsmanager.amazonaws.com AWS::SecretsManager::Secret/cnpg/xplane-image-gallery/roles/image-gallery-app — DeleteSecret by smana'
- kube_events (apps): ExternalSecret/xplane-image-gallery-cnpg-image-gallery UpdateFailed starting 17:31:51Z, repeated through 18:01:20Z: 'error processing spec.data[0] (key: cnpg/xplane-image-gallery/roles/image-gallery-app), err: operation error Secrets Manager: GetSecretValue, https response error StatusCode: 400, InvalidRequestException: You can't perform this operation on the secret because it was marked for deletion'
- pod_status (apps): all 3 xplane-image-gallery pods Pending with 'CreateContainerConfigError: secret "xplane-image-gallery-cnpg-image-gallery" not found'
- kube_events: Pod/xplane-image-gallery-* Failed events: 'Error: secret "xplane-image-gallery-cnpg-image-gallery" not found' (x141, x47, x22 across three pods)
- what_changed (apps/xplane-image-gallery): 'no changes found' — this is NOT a GitOps-triggered change; the deletion was out-of-band
- incident_timeline: chronological sequence shows [cloud] DeleteSecret at 17:31:26Z → [event] ExternalSecret UpdateFailed at 17:31:51Z → [event] Pod Failed at 17:51:22Z onward → alert at 18:03:30Z

## Cause

1. **A manual DeleteSecret operation by IAM user 'smana' at 2026-08-03T17:31:26Z deleted the AWS Secrets Manager secret 'cnpg/xplane-image-gallery/roles/image-gallery-app'. This secret is the source for the ExternalSecret 'xplane-image-gallery-cnpg-image-gallery' in the apps namespace, which populates the Kubernetes Secret of the same name that the image-gallery pods mount as DATABASE_URL credentials. With the AWS secret marked for deletion, the ExternalSecret cannot sync (InvalidRequestException on every reconciliation since 17:31:51Z), the K8s Secret is never created, and all pods sit in CreateContainerConfigError with 0 available replicas.** (92%) — change: CloudTrail DeleteSecret by smana at 2026-08-03T17:31:26Z on AWS::SecretsManager::Secret/cnpg/xplane-image-gallery/roles/image-gallery-app

## Resolution

- Restore the deleted AWS Secrets Manager secret immediately: 'aws secretsmanager restore-secret --secret-id cnpg/xplane-image-gallery/roles/image-gallery-app'. Since DeleteSecret marks the secret for deletion with a recovery window (not immediate permanent deletion), restore-secret should recover it. Once restored, the ExternalSecret controller will sync it on the next reconciliation cycle (typically within 1 minute), populating the K8s Secret. Then restart the Deployment: 'kubectl rollout restart deploy -n apps xplane-image-gallery'. Also re-trigger the failed AtlasMigration if needed. (reversible=true)

## Unresolved

- Why did user 'smana' manually delete the AWS Secrets Manager secret 'cnpg/xplane-image-gallery/roles/image-gallery-app'? This was a deliberate out-of-band action — the intent (accidental vs. intentional cleanup, wrong secret targeted, etc.) can only be determined by the human who performed it. The investigation confirms WHAT happened and HOW to fix it, but not the WHY behind the manual deletion.

## Citations

[1] CloudTrail DeleteSecret by smana at 2026-08-03T17:31:26Z on AWS::SecretsManager::Secret/cnpg/xplane-image-gallery/roles/image-gallery-app

