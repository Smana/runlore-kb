---
type: Incident
title: Application/<app> Degraded
description: The Argo CD Application `<app>` is Degraded because the underlying application container (`coder` in pod `<app>-<pod-id>`) is failing its readiness probes and continually restarting. This indicates an issue within the application itself causing it to be unhealthy.
resource: argocd/<app>
tags:
    - runlore
    - incident
timestamp: "2026-06-26T09:54:49Z"
fingerprint: ae64277ca3de8a2863a5522b31a6a9dae40bb07ff545c23c92e53fd7b3ba382c
---

## Decision

- **why keep:** The Argo CD Application `<app>` is Degraded because the underlying application container (`coder` in pod `<app>-<pod-id>`) is failing its readiness probes and continually restarting. This indicates an issue within the application itself causing it to be unhealthy.
- **confidence:** 80%

## Symptom

Application/<app> Degraded

## Investigate

- `pod_status` shows `<app>-<pod-id>` is `Running ready=0/1`.
- `kube_events` shows `Warning Pod/<app>-<pod-id> Unhealthy: Readiness probe failed: Get "http://10.146.191.144:8080/healthz": context deadline exceeded`.
- `kube_events` shows `Warning Pod/<app>-<pod-id> BackOff: Back-off restarting failed container coder`.
- `gitops_resource_status` shows Application `argocd/<app>` is `Ready=False (Degraded)` despite reporting "successfully synced (all tasks run)".

## Cause

1. **The Argo CD Application `<app>` is Degraded because the underlying application container (`coder` in pod `<app>-<pod-id>`) is failing its readiness probes and continually restarting. This indicates an issue within the application itself causing it to be unhealthy.** (80%)

## Resolution

- Investigate logs of the `coder` container in the `coder` namespace to identify the specific error causing the readiness probe failures and restarts. (reversible=true)

## Unresolved

- The exact reason for the application container's failure (e.g., specific error message from application logs) because `query_logs` tool was unavailable.
- The root cause of the "diff error: clone git@github.com:<org>/gitops-repo.git: invalid auth method" reported by `what_changed`, although it appears unrelated to the current symptoms given successful syncs.

