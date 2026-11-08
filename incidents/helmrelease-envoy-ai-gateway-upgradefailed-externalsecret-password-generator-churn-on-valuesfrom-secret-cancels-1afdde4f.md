---
type: Incident
title: HelmRelease envoy-ai-gateway UpgradeFailed — ExternalSecret Password-generator churn on valuesFrom Secret cancels…
description: 'The ExternalSecret ai-gateway-mcp-session-seed uses a Password generator with refreshPolicy: CreatedOnce. Each refresh generates a NEW random seed value and replaces the Secret ai-gateway-mcp-session-seed. Because that Secret is referenced as a valuesFrom source on the HelmRelease, and its template carries the label reconcile.fluxcd.io/watch: Enabled, Flux watches it. At 23:27:47Z the ExternalSecret refreshed and wrote a new Secret value; one second later (23:27:48Z) Flux detected the change and triggered a NEW HelmRelease reconciliation. That new reconciliation canceled the context of an in-progress Helm upgrade (which had been running only ~691ms), producing ''Helm upgrade failed ... context canceled''. The canceled upgrade left Helm revision v2 in status=failed, and after 1 upgrade failure Flux''s remediation exhausted its retries (Stalled, RetriesExceeded). The design is self-interfering: the same Secret that feeds chart values also churns on every Password-generator refresh and re-triggers the release it feeds.'
resource: envoy-ai-gateway-system/envoy-ai-gateway
tags:
    - runlore
    - incident
    - helmrelease
    - envoy-ai-gateway-system
timestamp: "2026-09-26T23:39:44Z"
fingerprint: 1afdde4f5fe7858880bb1446b73f6e5184b7d703c7727511c63ad832c1c7d67d
confidence: 0.82
---

## Decision

- **why keep:** The ExternalSecret ai-gateway-mcp-session-seed uses a Password generator with refreshPolicy: CreatedOnce. Each refresh generates a NEW random seed value and replaces the Secret ai-gateway-mcp-session-seed. Because that Secret is referenced as a valuesFrom source on the HelmRelease, and its template carries the label reconcile.fluxcd.io/watch: Enabled, Flux watches it. At 23:27:47Z the ExternalSecret refreshed and wrote a new Secret value; one second later (23:27:48Z) Flux detected the change and triggered a NEW HelmRelease reconciliation. That new reconciliation canceled the context of an in-progress Helm upgrade (which had been running only ~691ms), producing 'Helm upgrade failed ... context canceled'. The canceled upgrade left Helm revision v2 in status=failed, and after 1 upgrade failure Flux's remediation exhausted its retries (Stalled, RetriesExceeded). The design is self-interfering: the same Secret that feeds chart values also churns on every Password-generator refresh and re-triggers the release it feeds.
- **confidence:** 82%

## Symptom

HelmRelease envoy-ai-gateway UpgradeFailed — ExternalSecret Password-generator churn on valuesFrom Secret cancels in-progress Helm upgrade

Affected resource: HelmRelease envoy-ai-gateway-system/envoy-ai-gateway

## Investigate

- ExternalSecret ai-gateway-mcp-session-seed spec.dataFrom references a Password generator; spec.refreshPolicy=CreatedOnce; status.refreshTime=2026-09-26T23:27:47Z, status.conditions Ready=True (SecretSynced) (resource_spec)
- HelmRelease spec.valuesFrom references Secret/ai-gateway-mcp-session-seed valuesKey=seed targetPath=controller.mcp.sessionEncryption.seed; spec does NOT set upgrade.remediation.retries, so the default (0) applies — one failure stalls it (resource_spec)
- ExternalSecret target.template.metadata.labels includes reconcile.fluxcd.io/watch: Enabled, so Flux watches the generated Secret (resource_spec)
- kube_events at 23:27:47Z: 'ExternalSecret/ai-gateway-mcp-session-seed Created: secret created' — the Password generator wrote a new value (kube_events, all_types=true)
- HelmRelease status.conditions: Ready=False (HealthCheckCanceled, message 'New reconciliation triggered by Secret/envoy-ai-gateway-system/ai-gateway-mcp-session-seed'), Released=False (UpgradeFailed, 'Helm upgrade failed ... context canceled'), Stalled=True (RetriesExceeded, 'Failed to upgrade after 1 attempt(s)') (resource_spec)
- helm-controller logs at 23:27:48Z: 'release out-of-sync ... release config values changed' → 'running upgrade action with timeout of 5m0s' → 749ms later 'release is in a failed state' → 'New reconciliation triggered, canceling health checks' trigger=Secret/ai-gateway-mcp-session-seed; then at 23:27:58Z 'failed to wait for object to sync in-cache after patching ... context deadline exceeded' and 'Reconciler error ... terminal error: exceeded maximum retries: cannot remediate failed release' (controller_logs helm-controller envoy-ai-gateway)
- HelmRelease status.history shows v1 (status=deployed, lastDeployed 13:50:58Z) succeeded; v2 (status=failed, lastDeployed 23:27:48Z) failed; lastAttemptedReleaseActionDuration=691ms — the upgrade was canceled within a second, not a genuine timeout (resource_spec)
- resource_spec of HelmRelease: spec.install.remediation.retries=3 is present, but there is NO spec.upgrade block at all — so upgrade remediation defaults to 0 retries (resource_spec)
- HelmRelease status Stalled condition: reason=RetriesExceeded, message='Failed to upgrade after 1 attempt(s)' — a single failure was enough to stall (resource_spec)
- helm-controller log 23:27:58Z: 'Reconciler error ... terminal error: exceeded maximum retries: cannot remediate failed release' (controller_logs)
- KB runbook helmrelease-terminal-failed-exhausted-retries confirms: once retries are exhausted Flux stops retrying even after root cause is fixed; `flux reconcile helmrelease --reset` clears the counters (kb_search)

## Cause

1. **The ExternalSecret ai-gateway-mcp-session-seed uses a Password generator with refreshPolicy: CreatedOnce. Each refresh generates a NEW random seed value and replaces the Secret ai-gateway-mcp-session-seed. Because that Secret is referenced as a valuesFrom source on the HelmRelease, and its template carries the label reconcile.fluxcd.io/watch: Enabled, Flux watches it. At 23:27:47Z the ExternalSecret refreshed and wrote a new Secret value; one second later (23:27:48Z) Flux detected the change and triggered a NEW HelmRelease reconciliation. That new reconciliation canceled the context of an in-progress Helm upgrade (which had been running only ~691ms), producing 'Helm upgrade failed ... context canceled'. The canceled upgrade left Helm revision v2 in status=failed, and after 1 upgrade failure Flux's remediation exhausted its retries (Stalled, RetriesExceeded). The design is self-interfering: the same Secret that feeds chart values also churns on every Password-generator refresh and re-triggers the release it feeds.** (82%)
2. **The HelmRelease has entered a terminal stalled state (Stalled=True, RetriesExceeded, 'Failed to upgrade after 1 attempt(s)') because the HelmRelease spec defines spec.install.remediation.retries=3 but has NO spec.upgrade.remediation block, so upgrade remediation defaults to 0 retries. One canceled upgrade was enough to exhaust the (default-zero) upgrade retry budget, and Flux will not retry on its own. This is a secondary/config cause that amplified the first one — even if the Secret churn is fixed, Flux will stay Stalled until the counters are reset.** (70%)

## Resolution

- Two independent fixes are needed: (1) Stop the ExternalSecret from churning the value on every refresh — either set refreshPolicy to Merge (so the generated value persists across refreshes instead of being regenerated) or stop using a Password generator for a valuesFrom Secret that Flux watches; if a rotating seed is genuinely required, decouple it from the HelmRelease valuesFrom (mount it directly, not via the chart). (2) Clear the stalled Flux state: run `flux -n envoy-ai-gateway-system reconcile helmrelease envoy-ai-gateway --reset` to reset the exhausted upgrade failure counters and trigger a fresh install; if helm history shows only a failed revision, `helm -n envoy-ai-gateway-system uninstall envoy-ai-gateway` first, then the --reset reconcile. (reversible=true)
- Add spec.upgrade.remediation.retries to the HelmRelease (e.g. retries: 3) so a single transient upgrade failure does not immediately stall the release, then reset the existing stalled state with `flux -n envoy-ai-gateway-system reconcile helmrelease envoy-ai-gateway --reset`. (reversible=true)

## Unresolved

- Whether the MCP session seed is genuinely intended to be a rotating value (in which case the HelmRelease valuesFrom dependency should be removed and the seed mounted directly by the Deployment instead of passed through chart values) or whether CreatedOnce is a misconfiguration that should be Merge. This is a design decision for the chart owner.

