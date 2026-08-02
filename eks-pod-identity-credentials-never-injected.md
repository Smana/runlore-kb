---
type: Playbook
title: Pod crashloops with AWS "Access Denied" but the pod spec has no AWS_CONTAINER_* env — Pod Identity was never injected
description: An EKS Pod Identity workload fails every AWS call while the EPI, the PodIdentityAssociation, the IAM role and policy all report healthy. The pod was admitted before the association was usable, so the webhook injected no credentials — and because injection happens at admission, no restart, retry or backoff ever recovers it. Recreating the pod is the fix.
resource: pod://*
tags: [aws, eks, pod-identity, epi, iam, credentials, access-denied, crashloop, crossplane, webhook, admission, rebuild]
timestamp: 2026-08-02T00:00:00Z
last_validated: "2026-08-02"
---

# Symptom

- A pod that uses EKS Pod Identity crashloops on an AWS call it is clearly permitted to make:

  ```
  {"level":"fatal","error":"Access Denied.","message":"Failed to connect to storage"}
  ```

  The wording is application-specific — `Access Denied`, `NoCredentialProviders`,
  `Connect timeout on endpoint URL: 'http://169.254.170.23/v1/credentials'` — but the
  shape is constant: **AWS auth fails, and everything on the AWS side looks correct.**

- Restarts do not help. The container restarts on backoff indefinitely (36 restarts /
  ~3h in the observed case) with the identical error each time.

- Every check you would reach for passes:

  | Checked | Result |
  |---|---|
  | `EPI` XR | `Synced=True Ready=True` |
  | `PodIdentityAssociation` MR | `Ready=True`, correct `namespace` + `serviceAccount` |
  | IAM policy | present, correctly scoped to the right ARNs |
  | Target resource (bucket/table/queue) | exists, name matches the app's config |
  | Region, endpoint | match |

  This is what makes it expensive: the symptom points at AWS, and AWS is fine.

# Investigate

**The tell is in the POD SPEC, not in AWS.** Pod Identity works by the EKS webhook
mutating the pod at admission. Either it did, or it did not:

```bash
# 1. Are the credential env vars there?
kubectl get pod -n <ns> <pod> -o json \
  | jq -r '.spec.containers[0].env[]? | select(.name|test("AWS_CONTAINER")) | .name'

# 2. Is the projected token volume there?
kubectl get pod -n <ns> <pod> -o json \
  | jq -r '.spec.volumes[]? | select(.name=="eks-pod-identity-token") | .name'
```

A **healthy** pod-identity pod has all three:

```
AWS_CONTAINER_CREDENTIALS_FULL_URI=http://169.254.170.23/v1/credentials
AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE=/var/run/secrets/pods.eks.amazonaws.com/serviceaccount/eks-pod-identity-token
volume: eks-pod-identity-token
```

**Both commands returning empty confirms this playbook.** Nothing was injected, so the
AWS SDK never had credentials to send — which is why the IAM policy is irrelevant to the
failure.

Confirm the ordering, which is the actual cause:

```bash
kubectl get podidentityassociation.eks.aws.m.upbound.io -n <ns> <name> \
  -o jsonpath='{.metadata.creationTimestamp}{"  ready="}{range .status.conditions[?(@.type=="Ready")]}{.lastTransitionTime}{end}{"\n"}'
kubectl get pod -n <ns> <pod> -o jsonpath='{.metadata.creationTimestamp}{"\n"}'
```

If the pod's timestamp is at or before the association's, you have it.

# Root cause

Pod Identity credentials are injected **at pod admission**. A pod admitted before its
`PodIdentityAssociation` is usable gets nothing, and the pod spec is immutable
afterwards — so no restart, backoff or liveness kill ever recovers it. The pod is
permanently broken while every controller reports success.

Two distinct windows produce this:

1. **Creation ordering.** Something creates the workload and the association
   concurrently. Observed on a Crossplane composition that rendered the Deployment and
   the EPI in the same reconcile: one replica was admitted **2s before the
   `PodIdentityAssociation` object existed.**

2. **Webhook cache lag — the one you cannot design around.** `Ready=True` on the
   association does **not** mean the pod-identity webhook can serve it yet. The second
   replica in the same incident was admitted **1s after `Ready=True`** and still got
   nothing.

That second window is why ordering alone does not fix this, and why gating workload
creation on association readiness is not a solution: there is **no API to ask whether
the webhook is ready** — see the citations. A composition-level gate for exactly this was
tried and reverted (Smana/cloud-native-ref#1700, #1718).

Reproduces on **every cluster rebuild** and is invisible on steady-state clusters, where
the association long predates any pod.

# Resolution

Recreate the pods once the association exists. This is the upstream-prescribed remedy,
and the same one AWS documents for retrofitting Pod Identity onto Load Balancer
Controller, Cluster Autoscaler and Karpenter.

```bash
kubectl delete pod -n <ns> -l app.kubernetes.io/instance=<app>
```

For a Deployment/StatefulSet, `kubectl rollout restart` works equally well. Do **not**
bother touching IAM, the policy, or the association — they were never the problem.

# Verification

Do not stop at "the pod is Running" — confirm the injection actually happened:

```bash
# credentials now present
kubectl get pod -n <ns> <new-pod> -o json \
  | jq -r '.spec.containers[0].env[]? | select(.name|test("AWS_CONTAINER")) | "\(.name)=\(.value)"'

# and the app got past its AWS call
kubectl logs -n <ns> <new-pod> --tail=20
```

Observed after the fix: `Storage client initialized` in place of the fatal, and the
owning Crossplane XR moved to `Ready=True`.

# Prevention

- **After any cluster rebuild**, recreate pods for every EPI-backed workload rather than
  waiting for a failure report — the race is likely, not exceptional.
- **Do not gate workload creation on association readiness.** It cannot close the webhook
  window, and any mechanism that withholds a resource from the desired state also
  withholds it from `crossplane render`, breaking preview tooling.
- Ruled out and not worth re-testing: **`automountServiceAccountToken: false` is
  irrelevant.** A controlled admission test with the same ServiceAccount in the same
  namespace injects identically with it `false` or unset. The only variable that matters
  is *when* the pod was admitted.

# Citations

[1] aws/amazon-eks-pod-identity-webhook#264 — Pod Identity Association subject to a cache race condition
[2] aws/amazon-eks-pod-identity-webhook#174 — webhook not injecting env vars/volumes on initial deployment to a new cluster
[3] aws/containers-roadmap#2507 — no API to confirm whether a pod identity association is ready (open)
[4] Smana/cloud-native-ref#1700 — the incident; #1703 the attempted composition gate, #1718 its revert
