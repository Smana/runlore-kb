---
type: Incident
title: Kustomization flux-system/agent-sandbox HealthCheckFailed — HelmRelease agent-system/agent-sandbox UpgradeFailed:…
description: The HelmRelease agent-system/agent-sandbox is failing because the chart version string `0.1.0+527d9346fe1d` (SemVer build metadata) is used verbatim as a Kubernetes label value on the rendered ServiceMonitor (and other objects). Kubernetes labels forbid the `+` character — allowed chars are `[A-Za-z0-9], '-', '_', '.'`. The chart version `0.1.0+527d9346fe1d` was derived from the Git commit short SHA `527d9346fe1d` (the upstream GitRepository revision `v1.0.3@sha1:527d9346fe1d...`), and the chart templates inject this version into object labels (e.g. `agent-sandbox-{{ .Chart.Version }}`). The upgrade fails at the server-side-apply patch of the ServiceMonitor, surfacing as HelmRelease UpgradeFailed, which in turn stalls the parent Kustomization flux-system/agent-sandbox health check.
resource: agent-system/agent-sandbox
alert_resource: flux-system/agent-sandbox
tags:
    - runlore
    - incident
    - helmrelease
    - agent-system
timestamp: "2026-09-26T23:35:30Z"
fingerprint: c903eaa6b22937f5b1ed828785a2b9e2ba93b35af78c85ff7814f3a7759249a6
confidence: 0.84
provenance:
    - 'chart version 0.1.0+527d9346fe1d derived from Git revision v1.0.3@sha1:527d9346fe1d (commit ''examples(openclaw-fleet-gke): per-employee config ...'')'
---

## Decision

- **why keep:** The HelmRelease agent-system/agent-sandbox is failing because the chart version string `0.1.0+527d9346fe1d` (SemVer build metadata) is used verbatim as a Kubernetes label value on the rendered ServiceMonitor (and other objects). Kubernetes labels forbid the `+` character — allowed chars are `[A-Za-z0-9], '-', '_', '.'`. The chart version `0.1.0+527d9346fe1d` was derived from the Git commit short SHA `527d9346fe1d` (the upstream GitRepository revision `v1.0.3@sha1:527d9346fe1d...`), and the chart templates inject this version into object labels (e.g. `agent-sandbox-{{ .Chart.Version }}`). The upgrade fails at the server-side-apply patch of the ServiceMonitor, surfacing as HelmRelease UpgradeFailed, which in turn stalls the parent Kustomization flux-system/agent-sandbox health check.
- **confidence:** 84%
- **provenance:** chart version 0.1.0+527d9346fe1d derived from Git revision v1.0.3@sha1:527d9346fe1d (commit 'examples(openclaw-fleet-gke): per-employee config ...')

## Symptom

Kustomization flux-system/agent-sandbox HealthCheckFailed — HelmRelease agent-system/agent-sandbox UpgradeFailed: chart version contains invalid '+' character in label value

Affected resource: HelmRelease agent-system/agent-sandbox

## Investigate

- HelmRelease status (gitops_resource_status): Ready=False (UpgradeFailed), message: 'Helm upgrade failed for release agent-system/agent-sandbox with chart agent-sandbox@0.1.0+527d9346fe1d: failed to create resource: server-side apply failed for object agent-system/agent-sandbox-controller monitoring.coreos.com/v1, Kind=ServiceMonitor: ServiceMonitor.monitoring.coreos.com "agent-sandbox-controller" is invalid: metadata.labels: Invalid value: "agent-sandbox-0.1.0+527d9346fe1d": a valid label must be an empty string or consist of alphanumeric characters, \'-\', \'_\' or \'.', and must start and end with an alphanumeric character... regex used for validation is \'(([A-Za-z0-9][-A-Za-z0-9_.]*)?[A-Za-z0-9])?\'\''
- HelmChart source (gitops_resource_status): Ready=True (ChartPackageSucceeded), message: 'packaged agent-sandbox chart with version 0.1.0+527d9346fe1d' — the '+' is the offending character in the label value.
- GitRepository source (gitops_resource_status): Ready=True, revision 'v1.0.3@sha1:527d9346fe1d...' from https://github.com/kubernetes-sigs/agent-sandbox — the version suffix is the Git commit SHA, which is how '+' (SemVer build metadata separator) ended up in the chart version.
- Kustomization status (gitops_resource_status): Ready=False (HealthCheckFailed), message: 'health check failed after 5.024837501s: failed early due to stalled resources: [HelmRelease/agent-system/agent-sandbox status: \'Failed\']' — the Kustomization is healthy itself; it is stalled by the child HelmRelease.
- helm-controller logs (controller_logs): at 23:27:33Z 'release is in a failed state' followed by 'Reconciler error ... error: terminal error: missing target release for rollback: cannot remediate failed release' — the HelmRelease has entered a terminal state and will NOT auto-recover even after a fix.
- Last Helm logs (HelmRelease status): 'Error creating resource via patch: {"error":{},"gvk":"monitoring.coreos.com/v1, Kind=ServiceMonitor","name":"agent-sandbox-controller","namespace":"agent-system"}' — the ServiceMonitor is where the label validation rejects the value.
- helm-controller logs (controller_logs): 'Reconciler error ... error: terminal error: missing target release for rollback: cannot remediate failed release' at 2026-09-26T23:27:33Z
- KB runbook 'envoy-gateway HelmRelease terminal-failed' confirms: once retries are exhausted Flux stops retrying even after root cause is fixed; `flux reconcile helmrelease --reset` clears the counters

## Cause

1. **The HelmRelease agent-system/agent-sandbox is failing because the chart version string `0.1.0+527d9346fe1d` (SemVer build metadata) is used verbatim as a Kubernetes label value on the rendered ServiceMonitor (and other objects). Kubernetes labels forbid the `+` character — allowed chars are `[A-Za-z0-9], '-', '_', '.'`. The chart version `0.1.0+527d9346fe1d` was derived from the Git commit short SHA `527d9346fe1d` (the upstream GitRepository revision `v1.0.3@sha1:527d9346fe1d...`), and the chart templates inject this version into object labels (e.g. `agent-sandbox-{{ .Chart.Version }}`). The upgrade fails at the server-side-apply patch of the ServiceMonitor, surfacing as HelmRelease UpgradeFailed, which in turn stalls the parent Kustomization flux-system/agent-sandbox health check.** (84%) — change: chart version 0.1.0+527d9346fe1d derived from Git revision v1.0.3@sha1:527d9346fe1d (commit 'examples(openclaw-fleet-gke): per-employee config ...')
2. **Secondary: the HelmRelease has entered a terminal failed state. helm-controller logs show 'terminal error: missing target release for rollback: cannot remediate failed release' at 23:27:33Z. Even after the label-value fix is applied, a plain `flux reconcile` or `--force` may NOT be enough — the KB runbook (envoy-gateway incident) confirms Flux stops retrying once failure counters are exhausted and a `--reset` is required to clear counters and trigger a fresh install.** (40%)

## Resolution

- Fix the chart templates so the version used in labels strips the build-metadata portion after '+' (e.g. use {{ .Chart.Version | replace "+" "_" }} or {{ .Chart.AppVersion }} for labels), or pin the chart version to a plain SemVer without build metadata (drop the '+527d9346fe1d' suffix). Then clear the terminal failure with `flux reconcile helmrelease agent-sandbox -n agent-system --reset` (NOT just --force) because helm-controller already reports 'terminal error: ... cannot remediate failed release'. (reversible=true)
- After fixing the chart, run `flux reconcile helmrelease agent-sandbox -n agent-system --reset` to clear terminal failure counters. If helm release storage is wedged (helm history shows only failed revisions), `helm -n agent-system uninstall agent-sandbox` first, then the --reset reconcile. (reversible=true)

## Unresolved

- Whether the chart is vendored from the upstream kubernetes-sigs/agent-sandbox repo as-is (in which case the fix must be made upstream or via a post-render sidecar / values override) or maintained locally (in which case the template can be edited directly). The GitRepository points at https://github.com/kubernetes-sigs/agent-sandbox, suggesting upstream, but a human should confirm whether the cluster is meant to track upstream main or a fork.
- Whether the chart-version-from-commit-SHA behavior is intentional (a release pipeline choice) and whether stripping '+' will affect other consumers of the version label.

## Citations

[1] chart version 0.1.0+527d9346fe1d derived from Git revision v1.0.3@sha1:527d9346fe1d (commit 'examples(openclaw-fleet-gke): per-employee config ...')

