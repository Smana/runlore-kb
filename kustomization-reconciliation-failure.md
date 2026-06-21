---
type: Playbook
title: Flux Kustomization reconciliation failure
description: Diagnose a Flux Kustomization stuck Ready=False — build, dependency, or apply errors after a Git change.
resource: kustomization://*
tags: [flux, kustomization, gitops, reconcile, dependson, healthcheck]
timestamp: 2026-06-21T00:00:00Z
---

# Symptom

A Flux `Kustomization` reports `Ready=False`. The status message usually names the failing phase:
`build failed`, `dependency ... is not ready`, `health check failed`, or a server-side `apply` error
(immutable field, missing CRD, admission webhook rejection).

# Investigate

1. `what_changed_near(target, alert_time)` — find the Git revision the Kustomization is reconciling
   and the diff that landed (`flux-system` GitRepository → commit).
2. Read the Kustomization `status.conditions[Ready].message` verbatim — it points at the phase.
3. If `dependency ... not ready`: walk `spec.dependsOn` and find the upstream Kustomization that is
   itself `Ready=False` (the real root cause is upstream).
4. If `apply` error: the named object/field is the culprit (immutable spec change, CRD not installed
   yet, or a Kyverno/validating webhook denial).
5. Correlate with controller logs: `kustomize-controller` for build/apply, `source-controller` for
   the artifact fetch.

# Common causes

- A bad commit: invalid YAML, a `kustomization.yaml` referencing a deleted path, or a patch target
  that no longer matches.
- `dependsOn` upstream not ready (CRDs / Crossplane / namespaces not applied yet) — a cascade.
- Server-side apply rejecting an immutable field change (e.g. a Job/StatefulSet selector).
- A required CRD not installed (ordering) → `no matches for kind`.
- Admission webhook (Kyverno policy, PSS=restricted) rejecting the manifest.

# Resolution

- **Reversible first:** `flux suspend kustomization <name>` to stop the churn, or revert the offending
  commit so the source returns to a known-good revision.
- **Then fix forward:** correct the manifest / install the missing CRD / fix the dependency ordering,
  then `flux reconcile kustomization <name> --with-source`.

# Citations

- Flux Kustomization troubleshooting (upstream docs): conditions, dependsOn, health checks.
