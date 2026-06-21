---
type: Incident
title: Investigation into HarborCoreDown incident hampered by monitoring outage
description: 'Monitoring infrastructure outage: Metrics and log query tools are unable to connect to their respective backends. This prevents further investigation into the ''harbor'' workload''s actual state and might be the reason for the ''HarborCoreDown'' alert if the alert relies on these systems.'
tags:
    - runlore
    - incident
---

## Summary

Confidence 90%.

## Root causes
1. **Monitoring infrastructure outage: Metrics and log query tools are unable to connect to their respective backends. This prevents further investigation into the 'harbor' workload's actual state and might be the reason for the 'HarborCoreDown' alert if the alert relies on these systems.** (90%)
   - evidence: 'connection refused' errors when calling query_metrics and query_logs
   - suggested: Investigate and restore monitoring services (VictoriaMetrics/Prometheus and log backend). (reversible=true)

## Unresolved
- The actual operational status of the 'harbor' core.
- The specific reason for the 'HarborCoreDown' alert (if not related to monitoring).

