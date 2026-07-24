---
type: Incident
title: 'app-wizard new pods stuck NotReady: inverted /readyz readiness gate returns 503 when schema pipeline IS initialized'
description: Commit 3f5036e (image tag srcdiff-bug-3f5036e) introduced an inverted readiness gate in the /readyz HTTP handler in cmd/app-wizard/main.go. The new code checks `if pipeline != nil` and returns HTTP 503 (StatusServiceUnavailable) when the pipeline IS non-nil (i.e., initialized), and only returns 200 when pipeline IS nil. The comment says it should gate readiness on the schema pipeline being initialized BEFORE accepting traffic, but the logic is inverted — it blocks readiness exactly when the pipeline is ready. Since the app-wizard container starts and initializes its schema pipeline successfully (logs show 'app-wizard listening', 'LLM assists enabled'), `pipeline` becomes non-nil, the /readyz handler returns 503, the readiness probe fails, and the pod never becomes Ready.
resource: apps/app-wizard
tags:
    - runlore
    - incident
    - deployment
    - apps
timestamp: "2026-07-24T10:24:15Z"
fingerprint: 1932a5864b8a47efd32c1c1e5769d9b767ebfb138123490e2bdafff043404a47
confidence: 0.85
provenance:
    - github.com/Smana/app-wizard commit 3f5036e (image ghcr.io/smana/app-wizard:srcdiff-bug-3f5036e), file cmd/app-wizard/main.go, /readyz handler
---

## Decision

- **why keep:** Commit 3f5036e (image tag srcdiff-bug-3f5036e) introduced an inverted readiness gate in the /readyz HTTP handler in cmd/app-wizard/main.go. The new code checks `if pipeline != nil` and returns HTTP 503 (StatusServiceUnavailable) when the pipeline IS non-nil (i.e., initialized), and only returns 200 when pipeline IS nil. The comment says it should gate readiness on the schema pipeline being initialized BEFORE accepting traffic, but the logic is inverted — it blocks readiness exactly when the pipeline is ready. Since the app-wizard container starts and initializes its schema pipeline successfully (logs show 'app-wizard listening', 'LLM assists enabled'), `pipeline` becomes non-nil, the /readyz handler returns 503, the readiness probe fails, and the pod never becomes Ready.
- **confidence:** 85%
- **provenance:** github.com/Smana/app-wizard commit 3f5036e (image ghcr.io/smana/app-wizard:srcdiff-bug-3f5036e), file cmd/app-wizard/main.go, /readyz handler

## Symptom

app-wizard new pods stuck NotReady: inverted /readyz readiness gate returns 503 when schema pipeline IS initialized

Affected resource: Deployment apps/app-wizard

## Investigate

- source_diff f0a8361..3f5036e shows the offending hunk in cmd/app-wizard/main.go: `mux.HandleFunc("GET /readyz", func(w http.ResponseWriter, r *http.Request) { if pipeline != nil { http.Error(w, "schema pipeline not initialized", http.StatusServiceUnavailable); return } ... })` — the condition `pipeline != nil` returns 503, which is backwards; it should be `pipeline == nil`
- Commit message: 'test(runlore): inject inverted /readyz gate to validate source_diff (REVERT)' — the commit itself describes the gate as 'inverted'
- kube_events: Pod/app-wizard-6c46b7cdcd-f48xm Unhealthy (x10): Readiness probe failed: HTTP probe failed with statuscode: 503 — the 503 matches the http.StatusServiceUnavailable returned by the inverted gate
- pod_status: app-wizard-6c46b7cdcd-f48xm Running ready=4/5 — only the app-wizard container is not ready; all 4 sidecar containers (function-auto-ready, function-environment-configs, function-kcl, fetch-repo) are ready
- metrics kube_pod_container_status_ready: app-wizard container in pod 6c46b7cdcd-f48xm stays at ready=0 from 10:21:08 onward (flat 0>0>0>0>...), while the same container in the old pod 75b6d6547d-ttwbd is ready=1 — confirming the new image's /readyz handler is the cause
- Old pod ran image ghcr.io/smana/app-wizard:main-f0a8361 (ready=1, healthy); new pod runs ghcr.io/smana/app-wizard:srcdiff-bug-3f5036e (ready=0, stuck) — the regression maps exactly to the image bump
- app-wizard container logs show normal startup ('app-wizard listening', 'auth mode: github', 'LLM assists enabled') with no errors — the container is running fine, only the readiness probe is artificially failing

## Cause

1. **Commit 3f5036e (image tag srcdiff-bug-3f5036e) introduced an inverted readiness gate in the /readyz HTTP handler in cmd/app-wizard/main.go. The new code checks `if pipeline != nil` and returns HTTP 503 (StatusServiceUnavailable) when the pipeline IS non-nil (i.e., initialized), and only returns 200 when pipeline IS nil. The comment says it should gate readiness on the schema pipeline being initialized BEFORE accepting traffic, but the logic is inverted — it blocks readiness exactly when the pipeline is ready. Since the app-wizard container starts and initializes its schema pipeline successfully (logs show 'app-wizard listening', 'LLM assists enabled'), `pipeline` becomes non-nil, the /readyz handler returns 503, the readiness probe fails, and the pod never becomes Ready.** (85%) — change: github.com/Smana/app-wizard commit 3f5036e (image ghcr.io/smana/app-wizard:srcdiff-bug-3f5036e), file cmd/app-wizard/main.go, /readyz handler

## Resolution

- Revert the image bump to the previous version (ghcr.io/smana/app-wizard:main-f0a8361) in GitOps, or fix the inverted condition in cmd/app-wizard/main.go from `if pipeline != nil` to `if pipeline == nil` and deploy a new image. The commit message itself says '(REVERT)', indicating this was a test commit that should be reverted. (reversible=true)

## Unresolved

- The exact GitOps manifest/location that references the app-wizard image tag could not be identified (no Kustomization/Application was found via the available tools), so a human must locate and revert the image tag there

## Citations

[1] github.com/Smana/app-wizard commit 3f5036e (image ghcr.io/smana/app-wizard:srcdiff-bug-3f5036e), file cmd/app-wizard/main.go, /readyz handler

