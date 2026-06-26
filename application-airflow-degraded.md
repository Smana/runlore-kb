---
type: Incident
title: Application/airflow Degraded
description: Argo CD is failing to synchronize the airflow application due to an "invalid auth method" when attempting to clone the Git repository git@github.com:Aqemia/engineering.git.
resource: argocd/airflow
tags:
    - runlore
    - incident
timestamp: "2026-06-26T09:55:24Z"
fingerprint: bfaf826c82c774248efa8bf97ef4e4ac4a3395cc4c3a64acbbdf0b58c02105b4
---

## Decision

- **why keep:** Argo CD is failing to synchronize the airflow application due to an "invalid auth method" when attempting to clone the Git repository git@github.com:Aqemia/engineering.git.
- **confidence:** 95%

## Symptom

Application/airflow Degraded

## Investigate

- what_changed output for argocd/airflow: "diff error: clone git@github.com:Aqemia/engineering.git: invalid auth method"
- gitops_resource_status for argocd/airflow: Ready=False (Degraded SyncFailed), message: one or more synchronization tasks completed unsuccessfully (retried 5 times)., and repoURL: git@github.com:Aqemia/engineering.git

## Cause

1. **Argo CD is failing to synchronize the airflow application due to an "invalid auth method" when attempting to clone the Git repository git@github.com:Aqemia/engineering.git.** (95%)

## Resolution

- Investigate and correct the Git authentication method (e.g., SSH key or Git credential Secret) used by Argo CD for the git@github.com:Aqemia/engineering.git repository. After correcting the authentication, reconcile the argocd/airflow Application. (reversible=true)

## Unresolved

- The specific Git diff that triggered the synchronization attempt, as what_changed failed to retrieve it due to the authentication error.
- More detailed synchronization logs from the Argo CD application controller, as query_logs failed with a host lookup error.

