---
type: Incident
title: Kustomization eval-victim-broken ArtifactFailed — spec.path points to non-existent path in healthy source
description: Kustomization flux-system/eval-victim-broken has spec.path set to 'this/path/does/not/exist', a path that does not exist inside its source ExternalArtifact (infra-artifact). The kustomize-controller unpacks the artifact to /tmp/kustomization-<id>/ and stat()s the configured path, which fails with 'no such file or directory' on every reconcile (every ~1m), producing the ArtifactFailed condition.
resource: flux-system/eval-victim-broken
tags:
    - runlore
    - incident
    - kustomization
    - flux-system
timestamp: "2026-08-02T11:04:47Z"
fingerprint: 0604fa650496da6486540d97b2f733b1963804bd62229b9282417b877772b95c
confidence: 0.88
provenance:
    - none — what_changed returned 'no changes found'; the path is a static misconfiguration, not a recent diff
---

## Decision

- **why keep:** Kustomization flux-system/eval-victim-broken has spec.path set to 'this/path/does/not/exist', a path that does not exist inside its source ExternalArtifact (infra-artifact). The kustomize-controller unpacks the artifact to /tmp/kustomization-<id>/ and stat()s the configured path, which fails with 'no such file or directory' on every reconcile (every ~1m), producing the ArtifactFailed condition.
- **confidence:** 88%
- **provenance:** none — what_changed returned 'no changes found'; the path is a static misconfiguration, not a recent diff

## Symptom

Kustomization eval-victim-broken ArtifactFailed — spec.path points to non-existent path in healthy source

Affected resource: Kustomization flux-system/eval-victim-broken

## Investigate

- gitops_resource_status: Ready=False (ArtifactFailed), message 'kustomization path not found: stat /tmp/kustomization-1827285562/this/path/does/not/exist: no such file or directory'
- controller_logs (kustomize-controller): error='kustomization path not found: stat /tmp/kustomization-2540724022/this/path/does/not/exist...' repeated at 11:01:54 and 11:02:54
- gitops_tree: sourceRef ExternalArtifact/flux-system/infra-artifact is Ready=True — the source artifact is healthy and downloaded; the failure is purely the path, not the source

## Cause

1. **Kustomization flux-system/eval-victim-broken has spec.path set to 'this/path/does/not/exist', a path that does not exist inside its source ExternalArtifact (infra-artifact). The kustomize-controller unpacks the artifact to /tmp/kustomization-<id>/ and stat()s the configured path, which fails with 'no such file or directory' on every reconcile (every ~1m), producing the ArtifactFailed condition.** (88%) — change: none — what_changed returned 'no changes found'; the path is a static misconfiguration, not a recent diff

## Resolution

- Either (a) correct spec.path to a real directory that exists in the infra-artifact, or (b) if this is a test/eval fixture (the path 'this/path/does/not/exist' and name 'eval-victim-broken' suggest so), suspend or remove the Kustomization: 'flux suspend kustomization eval-victim-broken -n flux-system' to stop the reconciliation churn. (reversible=true)

## Unresolved

- Whether 'this/path/does/not/exist' is an intentional test fixture (the naming strongly suggests so) or a real misconfiguration that should point to a valid directory — a human who knows the repo layout should confirm the intended path or whether this Kustomization should exist at all.

## Citations

[1] none — what_changed returned 'no changes found'; the path is a static misconfiguration, not a recent diff

