---
type: Incident
title: HarborRegistryDown
description: 'The Harbor registry is down because the Crossplane resource responsible for provisioning its credentials (`AccessKey/xplane-harbor`) has hit an AWS IAM quota limit (`AccessKeysPerUser: 2`). This prevents the creation of a new access key, leaving the Kubernetes Secret (`xplane-harbor-access-key`) without the required `username` key, which in turn causes the `harbor-registry` pod to fail with a `CreateContainerConfigError`.'
tags:
    - runlore
    - incident
---

## Summary

Confidence 95%.

## Root causes
1. **The Harbor registry is down because the Crossplane resource responsible for provisioning its credentials (`AccessKey/xplane-harbor`) has hit an AWS IAM quota limit (`AccessKeysPerUser: 2`). This prevents the creation of a new access key, leaving the Kubernetes Secret (`xplane-harbor-access-key`) without the required `username` key, which in turn causes the `harbor-registry` pod to fail with a `CreateContainerConfigError`.** (95%)
   - evidence: pod_status shows harbor-registry pod failing with `CreateContainerConfigError: couldn't find key username in Secret tooling/xplane-harbor-access-key`.
   - evidence: kube_events for the `tooling` namespace shows a persistent Warning on the `AccessKey/xplane-harbor` resource: `LimitExceeded: Cannot exceed quota for AccessKeysPerUser: 2`.
   - evidence: The failed Crossplane `AccessKey` resource is responsible for creating the Secret used by the `harbor-registry` pod.
   - suggested: An administrator with AWS access should navigate to the IAM user associated with the Crossplane provider and delete an old or unused access key. This will allow Crossplane to create the new key required by Harbor. After the key is deleted, the `AccessKey/xplane-harbor` resource should be reconciled. (reversible=false)

## Unresolved
- The name of the specific IAM user that has reached its access key quota. This needs to be identified in the AWS console.
- Which of the two existing access keys is safe to delete.

