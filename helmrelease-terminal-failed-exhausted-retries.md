---
type: Playbook
title: HelmRelease stuck terminal-failed — exhausted retries after a transient install timeout
description: A Flux HelmRelease stays Ready=False ("exceeded maximum retries: cannot install/remediate release") long after the underlying cause cleared, because the install/upgrade failure counters are exhausted and Flux won't retry on its own.
resource: helmrelease://*
tags: [flux, helmrelease, helm, install, timeout, retries, remediation, reset, capacity]
timestamp: 2026-06-21T00:00:00Z
---

# Symptom

- A Flux `HelmRelease` is `Ready=False` with a stale message like
  `Helm install failed … timeout waiting for: [StatefulSet/… status: 'InProgress']`.
- helm-controller logs show the **terminal** state:
  `terminal error: exceeded maximum retries: cannot remediate failed release` or
  `… cannot install release` (`release not installed: no release in storage`).
- The underlying workload may **already be healthy** (or would be, given capacity) — Flux just
  won't try again. A plain `flux reconcile` and even `flux suspend`/`resume` do **not** clear it.
- Often hits many HelmReleases at once when a cluster-wide event (e.g. no schedulable nodes) made
  several installs time out together.

# Why it happens

Flux bounds install/upgrade attempts with `spec.install.remediation.retries` (and
`spec.upgrade.remediation.retries`). When the chart's `--wait` times out — e.g. pods stay `Pending`
because the cluster can't schedule them (see [[karpenter-ec2nodeclass-ami-not-found]]) — each attempt
counts as a failure. Once retries are exhausted the HelmRelease enters a **terminal** state and Flux
stops retrying, *even after* the root cause is fixed, because the failure counters persist in
`status`.

# Investigate

1. `kubectl get hr -A | grep -i false` — list the stuck releases.
2. `kubectl -n <ns> get hr <name> -o jsonpath='{.status.conditions[?(@.type=="Ready")].message}'`.
3. helm-controller logs for the terminal error:
   `kubectl -n flux-system logs -l app=helm-controller --tail=300 | grep -i <name>`
   → look for `exceeded maximum retries`.
4. Check the real workload state — it may be fine now:
   `kubectl -n <ns> get statefulset,deploy` and `helm -n <ns> history <name>`.
5. Confirm the original cause is resolved (capacity/nodes, dependency, config) before retrying.

# Resolution

- **Reset the failure counters and let Flux reinstall** (the key step):
  ```
  flux -n <ns> reconcile helmrelease <name> --reset
  ```
  `--reset` clears the install/upgrade failure counts, so Flux performs a fresh attempt; with the
  root cause gone, the install succeeds (`Helm install succeeded …`, `Ready=True`).
- If the helm release storage itself is wedged in a `failed` state (`helm -n <ns> history <name>`
  shows only `failed` with no good revision), `helm -n <ns> uninstall <name>` first, then
  `flux reconcile helmrelease <name> --reset`. StatefulSet PVCs persist across uninstall.
- Each `--reset` is a single reversible reconcile, but **bulk resets across shared/production
  namespaces should be human-approved** — scope and run them deliberately, not in one blind sweep.

# Prevention

- Raise `spec.install.timeout` / `spec.upgrade.timeout` for slow-starting charts so transient
  scheduling delays don't burn all retries.
- Fix capacity/autoscaling first (a single root like a broken Karpenter NodeClass can knock out many
  HelmReleases at once via simultaneous install timeouts).

# Citations

- Flux `HelmRelease` remediation + `flux reconcile helmrelease --reset` (upstream flux2 docs).
