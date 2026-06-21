---
type: Playbook
title: Karpenter EC2NodeClass not ready — AMI alias not found
description: Karpenter provisions no nodes; EC2NodeClass stays Ready=Unknown ("failed to discover any AMIs for alias"). Pods sit Pending on Insufficient cpu/memory and the cluster cannot scale.
resource: ec2nodeclass://*
tags: [karpenter, ec2nodeclass, nodepool, bottlerocket, ami, scaling, pending, insufficient-cpu, aws]
timestamp: 2026-06-21T00:00:00Z
---

# Symptom

- New/Pending pods never schedule: `FailedScheduling … Insufficient cpu` (or memory) on every existing node, and **no new node appears**.
- `kubectl get ec2nodeclass` shows `READY=Unknown`; the NodePool is `Ready=False`.
- Karpenter never creates a NodeClaim; the provisioner logs `ignoring nodepool, not ready`.
- Downstream blast radius: anything needing a fresh node stays Pending — e.g. VictoriaLogs/VictoriaTraces StatefulSets, observability, new workloads — which then surfaces as *its own* alerts (logs/traces unavailable).

# Investigate

1. `kubectl get ec2nodeclass` → look for `READY=Unknown/False`.
2. `kubectl get ec2nodeclass <name> -o jsonpath='{.status.conditions}'` → the failing condition is usually `AMIsReady` / `ValidationSucceeded` = `Awaiting AMI, Instance Profile, Security Group, and Subnet resolution`.
3. **Read the nodeclass-controller logs** — this carries the precise error:
   ```
   kubectl -n karpenter logs -l app.kubernetes.io/name=karpenter --tail=2000 \
     | grep -i 'nodeclass\|ami\|discover'
   ```
   Look for: `getting amis … failed to discover any AMIs for alias (alias=bottlerocket@X.Y.Z)`.
4. Confirm the alias version is invalid for the cluster's K8s version by checking the public SSM parameter:
   ```
   aws ssm get-parameter --region <region> \
     --name /aws/service/bottlerocket/aws-k8s-<k8s-ver>/x86_64/<bottlerocket-ver>/image_id
   ```
   An empty/NotFound result means that Bottlerocket version was never published for that K8s version → Karpenter finds no AMI.

# Common causes

- **EC2NodeClass pins a Bottlerocket alias version that doesn't exist for the cluster's K8s version** (e.g. `bottlerocket@1.54.0` on `aws-k8s-1.36`). Frequent after an EKS minor upgrade where the AMI alias wasn't bumped.
- Less common: AMI/SG/subnet selector tags match nothing; missing IAM perms for the nodeclass controller (`ec2:DescribeImages`, `ssm:GetParameter`).

# Resolution

- **Reversible / fix-forward via GitOps:** bump the EC2NodeClass `amiSelectorTerms[].alias` to a version that exists for the cluster's K8s version (verify with the SSM check above), or use `bottlerocket@latest`. Match the running nodes' version for reproducibility.
  ```yaml
  spec:
    amiSelectorTerms:
      - alias: bottlerocket@<valid-version>   # e.g. 1.62.0 for aws-k8s-1.36
  ```
- Reconcile the source; within ~1 min the EC2NodeClass goes `Ready=True`, Karpenter provisions NodeClaims, and the Pending pods schedule.

# Verification

- `kubectl get ec2nodeclass` → `READY=True`.
- `kubectl get nodeclaims` → new claims `Ready` shortly after.
- Previously-Pending pods (e.g. `victoria-logs-…-0`) transition to `Running`.

# Citations

- Karpenter `EC2NodeClass` AMI selection / `alias` (upstream karpenter-provider-aws docs).
- Bottlerocket image SSM parameter paths (`/aws/service/bottlerocket/aws-k8s-<ver>/…`).
