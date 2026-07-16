---
type: Playbook
title: EKS managed control plane — KubeAPIDown / KubeControllerManagerDown / KubeSchedulerDown are false positives
description: Three critical kube-prometheus *Down alerts fire continuously (usually right after cluster bring-up) because the EKS control plane is a managed service that is never scraped. They are structural false positives, not an outage — do not investigate the "down" component; silence the rule groups.
resource: vmrule://*
tags: [victoriametrics, vmalert, alerts, eks, control-plane, false-positive, kube-prometheus, kubeapidown, kubecontrollermanagerdown, kubeschedulerdown, observability]
timestamp: 2026-07-16T00:00:00Z
---

# Symptom

- Three `critical` alerts fire and stay firing, typically from the moment a cluster is built:
  `KubeAPIDown`, `KubeControllerManagerDown`, `KubeSchedulerDown`.
- Alert groups: `kubernetes-system-apiserver`, `kubernetes-system-controller-manager`,
  `kubernetes-system-scheduler`. Expressions are `absent(up{job="apiserver"})`,
  `absent(up{job="kube-controller-manager"})`, `absent(up{job="kube-scheduler"})`.
- The cluster is otherwise healthy — `kubectl` works, workloads schedule. That mismatch (API
  "down" but `kubectl` fine) is the tell that this is a scrape-absence artifact, not an outage.

# Investigate

1. Confirm nothing is actually scraped for those jobs:
   ```
   count by (job) (up{job=~"apiserver|kube-controller-manager|kube-scheduler|kube-proxy|kubelet"})
   ```
   On EKS you get **only `kubelet`**. `apiserver`, `kube-controller-manager`, `kube-scheduler`
   (and `kube-proxy`, replaced by Cilium here) have **no `up` series at all** → `absent()` is
   permanently true → the rule fires forever.
2. Verify scraping is intentionally off in the VM stack values
   (`observability/base/victoria-metrics-k8s-stack/vm-common-helm-values-configmap.yaml`):
   `kubeApiServer`, `kubeControllerManager`, `kubeScheduler`, `kubeEtcd`, `kubeProxy` all
   `enabled: false` ("managed service on EKS").
3. Note the gap: disabling the *scrape* does **not** disable the default *rule groups* — the
   `*Down` rules are generated independently and keep evaluating `absent(up{...})`.

# Root cause

The kube-prometheus default rule groups `kubernetes-system-{apiserver,controller-manager,scheduler}`
are rendered regardless of whether the corresponding component is scraped. On EKS the control plane
is AWS-managed and not exposed for scraping (and this cluster uses Cilium's kube-proxy replacement),
so those jobs never produce `up` series and the `absent()`-based `*Down` alerts fire structurally.

# Resolution

- In `victoria-metrics-k8s-stack` (0.86.0 uses the rule **sync-job** architecture) disable the
  groups in the stack values:
  ```yaml
  defaultRules:
    groups:
      kubernetes-system-apiserver:
        create: false
      kubernetes-system-controller-manager:
        create: false
      kubernetes-system-scheduler:
        create: false
  ```
  (Toggle is `defaultRules.groups.<group-name>.create=false`, **not** the older
  `defaultRules.rules.<key>` booleans — the chart maps by exact group name.)
- **Gotcha:** the sync-job does *not* garbage-collect groups that become disabled. On an existing
  cluster the already-created VMRules linger, so delete them once (they will not be recreated —
  the config now disables them, and they are `managed-by: sync-job`, not helm):
  ```
  kubectl delete vmrule -n observability \
    victoria-metrics-k8s-stack-rule-kubernetes-system-apiserver \
    victoria-metrics-k8s-stack-rule-kubernetes-system-controller-manager \
    victoria-metrics-k8s-stack-rule-kubernetes-system-scheduler
  ```
  A freshly-built cluster never creates them, so no manual step is needed there.

# Verification

- `kubectl get vmrule -n observability | grep -E 'kubernetes-system-(apiserver|controller-manager|scheduler)'`
  → no results.
- The compiled rulefiles ConfigMap `vm-victoria-metrics-k8s-stack-rulefiles-0` has 0 references to
  `KubeAPIDown|KubeControllerManagerDown|KubeSchedulerDown`.
- The `ALERTS{alertname=~"KubeAPIDown|…"}` series clears after VMAlert reloads (mounted-ConfigMap
  propagation + `-rule.configCheckInterval`, ~1–2 min); no pod restart required.

# Citations

- kube-prometheus-stack / victoria-metrics-k8s-stack `defaultRules` group toggles.
- EKS managed control plane is not scrapable; kube-proxy replaced by Cilium on this platform.
