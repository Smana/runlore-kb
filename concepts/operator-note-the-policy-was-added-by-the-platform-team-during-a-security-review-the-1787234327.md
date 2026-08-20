---
type: Concept
title: 'Operator note: the policy was added by the platform team during a security review; the…'
description: 'Operator knowledge from @U07JY4EQNDN via slack, on the finding "CheckoutUiUnavailable": the policy was added by the platform team during a security review; the intended fix is an allow rule for app=checkout-ui on TCP/9898'
resource: demo/checkout-ui
tags:
    - operator-note
    - slack
timestamp: "2026-08-20T13:58:48Z"
---

### 📝 Operator note

From **@U07JY4EQNDN** via slack on 2026-08-20T13:58:47Z.
Thread: CheckoutUiUnavailable

the policy was added by the platform team during a security review; the intended fix is an allow rule for app=checkout-ui on TCP/9898

### Context

- Corrects or extends: `incidents/pre-existing-2-checkoutuiunavailable-ciliumnetworkpolicy-still-denies-checkout-ui-orders-api-traffic-failing-81fe6337.md`
- Resource: `demo/checkout-ui`
- Trigger key: `checkoutuiunavailable|demo|deployment|checkout-ui|`


<!-- runlore-note:d6711613fb89b3eb -->
