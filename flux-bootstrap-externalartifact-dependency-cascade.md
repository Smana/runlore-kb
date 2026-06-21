---
type: Playbook
title: Flux bootstrap — Kustomizations DependencyNotReady until ArtifactGenerator artifacts exist
description: On a fresh cluster, Kustomizations whose source is an ArtifactGenerator-produced ExternalArtifact report DependencyNotReady (source "not found") until the generator runs. It cascades across dependents and self-resolves; it is not a deleted or misconfigured source.
tags:
    - flux
    - bootstrap
    - artifactgenerator
    - externalartifact
    - dependencynotready
    - cascade
    - source-controller
---

# Symptom

Shortly after a fresh `flux` bootstrap, many Kustomizations are `Ready=False` with
`DependencyNotReady`, and an investigation reports the *source* as missing — e.g.
`get ... infra-artifact ... not found`. The dependents fan out (e.g. `crossplane-configuration`,
`eks-pod-identities`, `security`, `karpenter`, `crds`), each blaming a missing `*-artifact` source.

# Why it happens (this platform)

The repo is split into per-area artifacts by a Flux **`ArtifactGenerator`** (`flux-system/monorepo-split`),
which produces **`ExternalArtifact`** objects (`infra-artifact`, `security-artifact`, `crds-artifact`,
`apps-artifact`, …). Kustomizations reference them via `spec.sourceRef.kind: ExternalArtifact`.

During the initial sync there is a window where the dependent Kustomizations exist but the
ArtifactGenerator has **not yet produced** the ExternalArtifacts — so the source is genuinely "not
found yet", dependents go `DependencyNotReady`, and it **cascades** down the `dependsOn` graph. Once
`monorepo-split` reconciles ("generated N artifact(s)"), the ExternalArtifacts appear and the whole
chain becomes Ready on its own. **This is a transient bootstrap race, not a deleted/misconfigured
source.**

# Investigate

1. `kubectl -n flux-system get artifactgenerator` → is `monorepo-split` `Ready` and "generated N
   artifact(s)"? If not yet, that's the bottleneck — check `kustomize-controller`/the generator.
2. `kubectl -n flux-system get externalartifact` → are the `*-artifact` objects present?
3. Confirm the dependents' `spec.sourceRef.kind` is `ExternalArtifact` (not `GitRepository`).
4. Walk `dependsOn` to the first not-ready node — the rest are cascade victims.

# Resolution

- **Usually none — wait.** Once the ArtifactGenerator generates the artifacts, the cascade clears
  itself. Re-check after a reconcile.
- If the generator is stuck: `flux -n flux-system reconcile kustomization flux-artifact-generators`
  (or whatever Kustomization owns the ArtifactGenerator), then reconcile the dependents.
- Do **not** "recreate the missing GitRepository" — there is no GitRepository; the source is an
  ExternalArtifact. Recreating the wrong kind is a dead end (see the gotcha below).

# Gotcha — "GitRepository not found" is often a tooling artifact

An investigation that only resolves `GitRepository` sources will report `gitrepository <x>-artifact
not found` even though the real source is an `ExternalArtifact` that exists. Treat "GitRepository not
found" for a `*-artifact` name as **wrong-kind lookup**, not a missing source — check
`get externalartifact` before concluding anything was deleted.

# Citations

- Flux `ArtifactGenerator` / `ExternalArtifact` (source-watcher / fluxcd extensions).
- Related: [[helmrelease-terminal-failed-exhausted-retries]], [[kustomization-reconciliation-failure]].
