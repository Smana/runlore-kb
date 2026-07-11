---
type: Incident
title: Kustomization broken-kustomization ArtifactFailed — non-existent spec.path (this-path-does-not-exist)
description: 'The Kustomization''s spec.path is set to ''this-path-does-not-exist'', a directory that does not exist in the podinfo Git repository. The kustomize-controller successfully fetches the GitRepository artifact (source is Ready=True) but fails when trying to stat the build path, producing the persistent error: ''kustomization path not found: stat /tmp/kustomization-.../this-path-does-not-exist: no such file or directory''. The alert''s ''Source artifact not found, retrying in 10s'' message is a transient message logged at 15:07:03 before the GitRepository had stored its artifact; by 15:07:04 the artifact was stored and the real, persistent failure became the non-existent path.'
resource: runlore-test/broken-kustomization
tags:
    - runlore
    - incident
    - kustomization
    - runlore-test
timestamp: "2026-07-11T15:10:25Z"
fingerprint: 2c419b4f8095c00468cbfdcf7bf6b5e073e627f36b388dc4899415acb5f08227
confidence: 0.9
---

## Decision

- **why keep:** The Kustomization's spec.path is set to 'this-path-does-not-exist', a directory that does not exist in the podinfo Git repository. The kustomize-controller successfully fetches the GitRepository artifact (source is Ready=True) but fails when trying to stat the build path, producing the persistent error: 'kustomization path not found: stat /tmp/kustomization-.../this-path-does-not-exist: no such file or directory'. The alert's 'Source artifact not found, retrying in 10s' message is a transient message logged at 15:07:03 before the GitRepository had stored its artifact; by 15:07:04 the artifact was stored and the real, persistent failure became the non-existent path.
- **confidence:** 90%

## Symptom

Kustomization broken-kustomization ArtifactFailed — non-existent spec.path (this-path-does-not-exist)

Affected resource: Kustomization runlore-test/broken-kustomization

## Investigate

- gitops_resource_status: Kustomization Ready=False (ArtifactFailed), message='kustomization path not found: stat /tmp/kustomization-344181676/this-path-does-not-exist: no such file or directory'
- gitops_tree: GitRepository runlore-test/podinfo-src is Ready=True — source is healthy
- gitops_resource_status (GitRepository): Ready=True (Succeeded), stored artifact for revision master@sha1:917cb24455d33b1c54fecf76669a81d14cae0ad3 from https://github.com/stefanprodan/podinfo
- source-controller logs at 15:07:04: 'stored artifact for commit Merge pull request #504...' — artifact fetch succeeds
- kustomize-controller logs: transient 'Source artifact not found, retrying in 10s' at 15:07:03 (before artifact stored), then persistent 'kustomization path not found' error at 15:07:04 and 15:08:05
- what_changed: 'no changes found' — no GitOps-managed change caused this; the Kustomization was created with this bad path
- incident_timeline: Kustomization and GitRepository both created at ~15:07; no preceding Git changes; cloud events are unrelated S3/KMS/SSM activity for an OpenBao snapshot bucket

## Cause

1. **The Kustomization's spec.path is set to 'this-path-does-not-exist', a directory that does not exist in the podinfo Git repository. The kustomize-controller successfully fetches the GitRepository artifact (source is Ready=True) but fails when trying to stat the build path, producing the persistent error: 'kustomization path not found: stat /tmp/kustomization-.../this-path-does-not-exist: no such file or directory'. The alert's 'Source artifact not found, retrying in 10s' message is a transient message logged at 15:07:03 before the GitRepository had stored its artifact; by 15:07:04 the artifact was stored and the real, persistent failure became the non-existent path.** (90%)

## Resolution

- Correct the Kustomization's spec.path to point to a directory that actually exists in the podinfo repository (e.g., './kustomize' or './deploy'), or delete the Kustomization if it is an intentional test resource. Then run 'flux reconcile kustomization broken-kustomization --with-source' to verify. (reversible=true)

## Unresolved

- Whether this is an intentionally-broken test/demo resource (naming strongly suggests it) — if so, the appropriate response may be to suppress the alert rather than fix the path. Only the namespace owner can confirm the intent.

