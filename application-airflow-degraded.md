---
type: Incident
title: Application/airflow Degraded
description: 'Missing ExternalSecret for database credentials: [REDACTED] ExternalSecret/wet-collab-database-admin-credentials is failing with "error processing spec.dataFrom[0].extract, err: Secret does not exist". This is a critical dependency for Airflow, likely preventing it from connecting to its database and thus failing synchronization tasks.'
resource: argocd/airflow
tags:
    - runlore
    - incident
timestamp: "2026-06-26T12:06:25Z"
fingerprint: 1c203f16cd77d6fc915e6affc8f4b1fbd63d7d27de3dd39a1f337e269ec89a94
---

## Decision

- **why keep:** Missing ExternalSecret for database credentials: [REDACTED] ExternalSecret/wet-collab-database-admin-credentials is failing with "error processing spec.dataFrom[0].extract, err: Secret does not exist". This is a critical dependency for Airflow, likely preventing it from connecting to its database and thus failing synchronization tasks.
- **confidence:** 90%

## Symptom

Application/airflow Degraded

## Investigate

- kube_events output: Warning ExternalSecret/wet-collab-database-admin-credentials UpdateFailed (x648): error processing spec.dataFrom[0].extract, err: Secret does not exist
- kube_events output: Warning Pod/airflow-redis-0 FailedAttachVolume: Multi-Attach error for volume "pvc-1c4bdaa1-af1e-43ed-9e22-6f6e0de36801" Volume is already exclusively attached to one node and can't be attached to another
- kube_events output: Warning Pod/airflow-scheduler-549b754f9-lmmfv FailedCreatePodSandBox: Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox ... plugin type="aws-cni" name="aws-cni" failed (add): add cmd: failed to setup network policy
- kube_events output: Warning Pod/airflow-scheduler-549b754f9-lmmfv FailedScheduling: 0/23 nodes are available: 1 Too many pods, 15 node(s) had untolerated taint(s), 3 Insufficient memory, 6 Insufficient cpu.

## Cause

1. **Missing ExternalSecret for database credentials: [REDACTED] ExternalSecret/wet-collab-database-admin-credentials is failing with "error processing spec.dataFrom[0].extract, err: Secret does not exist". This is a critical dependency for Airflow, likely preventing it from connecting to its database and thus failing synchronization tasks.** (90%)
2. **Redis volume attachment failure: The airflow-redis-0 pod is experiencing a FailedAttachVolume error. Redis is a critical component for Airflow's caching and message brokering.** (80%)
3. **Airflow scheduler pod failures: Airflow scheduler pods are encountering network setup failures and resource constraints, preventing them from being scheduled or starting correctly.** (70%)

## Resolution

- Investigate why the wet-collab-database-admin-credentials secret is missing or not accessible. Ensure the secret exists in the target namespace and the ExternalSecret resource is correctly configured to extract it. (reversible=false)
- Investigate the Multi-Attach error for the Redis PVC. This usually indicates an issue with the underlying storage provisioner or a Kubernetes bug where a volume is not correctly detached from a previous node. (reversible=false)
- Investigate the AWS CNI plugin failure for network setup and address the node resource constraints (too many pods, insufficient memory/CPU) and untolerated taints that prevent scheduler pods from being scheduled. (reversible=false)

## Unresolved

- Specific Git change causing the issue: what_changed failed to retrieve the diff due to an "invalid auth method" for git@github.com:Aqemia/engineering.git. Therefore, it's unclear if a recent Git change introduced the missing ExternalSecret or other configuration issues.

