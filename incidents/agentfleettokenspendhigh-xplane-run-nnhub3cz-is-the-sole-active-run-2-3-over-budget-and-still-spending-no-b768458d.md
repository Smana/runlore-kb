---
type: Incident
title: AgentFleetTokenSpendHigh — xplane-run-nnhub3cz is the sole active run, 2.3× over budget and still spending; no…
description: AgentRun xplane-run-nnhub3cz is the only run still in Running phase and is the dominant token consumer (3.512M tokens — ~35% of the fleet's 10.087M total). It has exceeded its budget_max_tokens of 1.5M by 2.34× and is still actively spending (+328K jump at 10:00, still rising). Revoking this run is the immediate mitigation to stop the bleed; it is the single noisiest run the alert names.
resource: agents/xplane-run-nnhub3cz
tags:
    - runlore
    - incident
    - agentrun
    - agents
timestamp: "2026-10-28T10:07:33Z"
fingerprint: b768458d3594992b395ad70bb7364232f187e336c8be9ac268f15dbe9e985c53
confidence: 0.93
---

## Decision

- **why keep:** AgentRun xplane-run-nnhub3cz is the only run still in Running phase and is the dominant token consumer (3.512M tokens — ~35% of the fleet's 10.087M total). It has exceeded its budget_max_tokens of 1.5M by 2.34× and is still actively spending (+328K jump at 10:00, still rising). Revoking this run is the immediate mitigation to stop the bleed; it is the single noisiest run the alert names.
- **confidence:** 93%

## Symptom

AgentFleetTokenSpendHigh — xplane-run-nnhub3cz is the sole active run, 2.3× over budget and still spending; no enforced per-run cap

Affected resource: AgentRun agents/xplane-run-nnhub3cz

## Investigate

- alert_rule: AgentFleetTokenSpendHigh expr `sum(increase(gen_ai_client_token_usage_sum{ar_agent=~"system:serviceaccount:agents:.*", gen_ai_token_type=~"input|output"}[1h])) > 1e7` — query_metrics confirms current value = 10.087M, exceeding the 10M threshold with for=5m hold-down satisfied
- query_metrics topk: xplane-run-nnhub3cz input tokens = 3.489M (next run a727jvn5 = 1.928M) — it alone is ~3× the next contributor
- query_metrics agentrun_status_phase{phase="Running"}==1 returns ONLY xplane-run-nnhub3cz — all other 22 runs are Succeeded/Failed/BudgetExhausted
- query_metrics_range agentrun_usage_tokens{name="xplane-run-nnhub3cz"}: trend 0>69156>219136>...>3.442M>3.477M>3.512M, biggest jump +328712@10:00:15Z — spend is still actively climbing
- query_metrics agentrun_usage_tokens/agentrun_budget_max_tokens for nnhub3cz = 2.3416 (3.512M used / 1.5M budget)
- pod_status: xplane-run-nnhub3cz Running ready=4/4 age=27m — the pod is alive and consuming tokens right now
- agentrun_info: nnhub3cz is repo=Smana/cloud-native-ref branch=agent/c3hzsaqk role=implementer tier=standard principal=system:factory
- alert_rule annotation: description='No enforced per-run cap exists until SP4 PR 2. Revoke the noisiest run by hand: kubectl annotate agentrun -n agents <name> agents.ogenki.io/revoked=budget-run'
- query_metrics topk(agentrun_usage_tokens / agentrun_budget_max_tokens): xplane-run-bajd7iz5 = 4.93 (7.39M used / 1.5M budget), xplane-run-fsiwi5m4 = 4.28 (1.28M / 300K) — both massively over budget, proving enforcement is absent not just for nnhub3cz
- query_metrics agentrun_status_phase: bajd7iz5 is in Failed phase (not BudgetExhausted) despite 4.93× overage — it was stopped by failure, not by budget enforcement
- query_metrics agentrun_status_phase: fsiwi5m4 reached BudgetExhausted but only after spending 4.28× its budget — enforcement, if any, is lagging/permissive

## Cause

1. **AgentRun xplane-run-nnhub3cz is the only run still in Running phase and is the dominant token consumer (3.512M tokens — ~35% of the fleet's 10.087M total). It has exceeded its budget_max_tokens of 1.5M by 2.34× and is still actively spending (+328K jump at 10:00, still rising). Revoking this run is the immediate mitigation to stop the bleed; it is the single noisiest run the alert names.** (93%)
2. **The AgentRun controller does not enforce the per-run budget cap — budget_max_tokens is advisory, not a hard stop. The alert annotation itself states 'No enforced per-run cap exists until SP4 PR 2'. Multiple runs exceeded their budgets before being stopped: xplane-run-bajd7iz5 spent 7.39M against a 1.5M budget (4.93×), and xplane-run-fsiwi5m4 spent 1.28M against a 300K budget (4.28×) before reaching BudgetExhausted phase. This systemic gap is why a single runaway implementer run could drive the entire fleet over 10M tokens/hour.** (55%)

## Resolution

- Revoke the run as the alert prescribes: kubectl annotate agentrun -n agents xplane-run-nnhub3cz agents.ogenki.io/revoked=budget-run. Then verify agentrun_status_phase transitions to Revoked and that gen_ai_client_token_usage stops increasing for ar_agent=system:serviceaccount:agents:xplane-run-nnhub3cz. (reversible=true)
- Land SP4 PR 2 to make budget_max_tokens an enforced hard cap that transitions a run to Revoked/BudgetExhausted when usage reaches the limit, rather than relying on manual intervention. Until then, this alert will recur whenever a run loops past its budget. (reversible=false)

## Unresolved

- Cannot read the AgentRun CR spec to confirm the exact budget field name and whether a 'revoked' annotation is the correct controller-recognized mechanism — this is inferred from the alert annotation, not verified against the CRD schema. The on-call should confirm the annotation key before applying it.

