---
type: Incident
title: headlamp-oauth2-proxy HelmRelease terminal-failed (InstallFailed) — root cause already fixed, needs HelmRelease…
description: The headlamp-oauth2-proxy HelmRelease is terminal-failed (Stalled=True, reason=RetriesExceeded, installFailures=4) after exhausting its 3 install retries during a window when its OIDC provider (ZITADEL) was not yet running. Flux will not retry on its own. The original root cause — ZITADEL not being available — is now fixed (ZITADEL HelmRelease Ready=True/InstallSucceeded at 08:28:42Z; headlamp-oauth2-proxy pod is Running ready=1/1 and serving traffic). What remains is the terminal-failed HelmRelease that needs `flux reconcile helmrelease --reset` to clear the exhausted failure counters and perform a fresh install.
resource: tooling/headlamp-oauth2-proxy
tags:
    - runlore
    - incident
    - helmrelease
    - tooling
timestamp: "2026-09-01T09:14:09Z"
fingerprint: 2a2e1fba43072673d680b2d5c97631e9f0e9140a256d8b5ed7c8d7ae262df947
confidence: 0.82
provenance:
    - No Git change to headlamp-oauth2-proxy; what_changed(tooling) shows only a stale crds-actions-runner-controller sync at 08:58. The failure was caused by a transient dependency (ZITADEL) not being ready during the install window.
---

## Decision

- **why keep:** The headlamp-oauth2-proxy HelmRelease is terminal-failed (Stalled=True, reason=RetriesExceeded, installFailures=4) after exhausting its 3 install retries during a window when its OIDC provider (ZITADEL) was not yet running. Flux will not retry on its own. The original root cause — ZITADEL not being available — is now fixed (ZITADEL HelmRelease Ready=True/InstallSucceeded at 08:28:42Z; headlamp-oauth2-proxy pod is Running ready=1/1 and serving traffic). What remains is the terminal-failed HelmRelease that needs `flux reconcile helmrelease --reset` to clear the exhausted failure counters and perform a fresh install.
- **confidence:** 82%
- **provenance:** No Git change to headlamp-oauth2-proxy; what_changed(tooling) shows only a stale crds-actions-runner-controller sync at 08:58. The failure was caused by a transient dependency (ZITADEL) not being ready during the install window.

## Symptom

headlamp-oauth2-proxy HelmRelease terminal-failed (InstallFailed) — root cause already fixed, needs HelmRelease reset (occurrence #2, pre-existing)

Affected resource: HelmRelease tooling/headlamp-oauth2-proxy

## Investigate

- resource_spec HelmRelease tooling/headlamp-oauth2-proxy: status.conditions Stalled=True reason=RetriesExceeded message='Failed to install after 4 attempt(s)', installFailures=4, history[0].status=failed, firstDeployed=07:50:21Z, spec.install.remediation.retries=3
- gitops_resource_status HelmRelease security/zitadel: Ready=True (InstallSucceeded) at 08:28:42Z — ZITADEL is now healthy, confirming the original root cause is fixed
- pod_status tooling: headlamp-oauth2-proxy-57d588f5f5-rlttm Running ready=1/1 restarts=13 last-exit=2026-09-01T08:27:02Z — the pod eventually became healthy after ZITADEL came up
- pod_logs (previous=true) headlamp-oauth2-proxy: 'ERROR: Failed to initialise OAuth2 Proxy: ... failed to discover OIDC configuration: error performing request: Get "https://auth.gcp.cloud.ogenki.io/.well-known/openid-configuration": read tcp 100.65.2.174:51618->35.204.2.200:443: read: connection reset by peer' — the crash-loop was caused by ZITADEL not being reachable
- kube_events security: Pod/zitadel-setup Failed (ImagePullBackOff): 'Failed to pull image "docker.io/alpine/k8s:1.35.7": rpc error: code = NotFound desc = ... docker.io/alpine/k8s:1.35.7: not found' — the ZITADEL setup Job's sidecar image was not found, blocking ZITADEL's own install (and thus oauth2-proxy's OIDC discovery) from 08:03–08:27
- kb_search matched runbook 'helmrelease-terminal-failed-exhausted-retries': confirms Flux stops retrying after retries exhausted; `flux reconcile helmrelease --reset` clears the counters for a fresh install
- pod_logs (current) headlamp-oauth2-proxy: pod is actively serving HTTP 200 responses to /ping, /ready, and headlamp.priv.gcp.ogenki.io requests — the service is functionally healthy despite the terminal-failed HelmRelease
- kube_events security: 'Failed to pull image "docker.io/alpine/k8s:1.35.7": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/alpine/k8s:1.35.7": failed to resolve reference "docker.io/alpine/k8s:1.35.7": docker.io/alpine/k8s:1.35.7: not found' (x4 ErrImagePull, x10 ImagePullBackOff across multiple Job pods: zitadel-setup-tmnww, zitadel-setup-rq6qj, zitadel-setup-5b5qf)
- kube_events security: Job/zitadel-setup DeadlineExceeded at 08:08:40, 08:14:32, 08:20:43, 08:27:42 — the Job exceeded its deadline because the sidecar couldn't pull its image
- gitops_resource_status HelmRelease security/zitadel: 4 InstallFailed events (08:08, 08:14, 08:20, 08:27) each 'failed pre-install: failed early due to stalled resources: [Job/security/zitadel-setup status: Failed]', then finally InstallSucceeded at 08:28:42Z
- pod_status security: zitadel-setup-6d8ft Succeeded ready=0/3 age=38m — the setup Job eventually succeeded (confirming the image issue resolved, possibly via cache or retry)

## Cause

1. **The headlamp-oauth2-proxy HelmRelease is terminal-failed (Stalled=True, reason=RetriesExceeded, installFailures=4) after exhausting its 3 install retries during a window when its OIDC provider (ZITADEL) was not yet running. Flux will not retry on its own. The original root cause — ZITADEL not being available — is now fixed (ZITADEL HelmRelease Ready=True/InstallSucceeded at 08:28:42Z; headlamp-oauth2-proxy pod is Running ready=1/1 and serving traffic). What remains is the terminal-failed HelmRelease that needs `flux reconcile helmrelease --reset` to clear the exhausted failure counters and perform a fresh install.** (82%) — change: No Git change to headlamp-oauth2-proxy; what_changed(tooling) shows only a stale crds-actions-runner-controller sync at 08:58. The failure was caused by a transient dependency (ZITADEL) not being ready during the install window.
2. **The deeper root cause that made ZITADEL unavailable was the zitadel-setup Job's sidecar image `docker.io/alpine/k8s:1.35.7` not being found (ImagePullBackOff → DeadlineExceeded), which caused ZITADEL's own HelmRelease to fail 4 times before finally succeeding at 08:28:42Z. This image-not-found issue may recur on future installs if the image tag is invalid or unavailable.** (55%)

## Resolution

- Run `flux -n tooling reconcile helmrelease headlamp-oauth2-proxy --reset` to clear the exhausted failure counters and let Flux perform a fresh install. The underlying root cause (ZITADEL OIDC provider) is already fixed — ZITADEL is Ready=True and the oauth2-proxy pod is Running. With the OIDC provider available, the fresh install should succeed immediately. Verify with `kubectl -n tooling get hr headlamp-oauth2-proxy` → Ready=True. (reversible=true)
- Investigate why `docker.io/alpine/k8s:1.35.7` was not found — the tag may not exist on Docker Hub (the latest alpine/k8s tags are typically 1.x.x without patch-level matching). Consider pinning to a valid image tag or using a mirror to prevent recurrence on future ZITADEL installs. This is a secondary issue since ZITADEL is now running. (reversible=true)

## Unresolved

- Why `docker.io/alpine/k8s:1.35.7` was not found during 08:03–08:27 but the zitadel-setup Job eventually succeeded at ~08:28 — the image may have become available/cached, or a different Job pod succeeded. This is a secondary issue (ZITADEL is now running) but could recur on future installs.

## Citations

[1] No Git change to headlamp-oauth2-proxy; what_changed(tooling) shows only a stale crds-actions-runner-controller sync at 08:58. The failure was caused by a transient dependency (ZITADEL) not being ready during the install window.

