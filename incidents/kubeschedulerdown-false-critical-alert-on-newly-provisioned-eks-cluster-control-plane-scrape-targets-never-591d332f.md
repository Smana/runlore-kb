---
type: Incident
title: KubeSchedulerDown — false critical alert on newly provisioned EKS cluster (control plane scrape targets never…
description: Newly provisioned EKS cluster deployed the VictoriaMetrics monitoring stack with default Kubernetes mixin alerting rules, but no scrape configuration for EKS control plane components (kube-scheduler, kube-controller-manager, apiserver). On EKS these run on AWS-managed infrastructure and are not exposed as Prometheus scrape targets by default, so the KubeSchedulerDown alert (along with KubeControllerManagerDown and KubeAPIDown) fires from the moment monitoring comes online.
resource: observability/victoria-metrics-k8s-stack
tags:
    - runlore
    - incident
    - vmalert/victoria-metrics-k8s-stack
    - observability
timestamp: "2026-07-13T10:25:28Z"
fingerprint: 591d332fdba0decc5a7517747d832009e32d877b0ad9265df67af41801a1a4f4
confidence: 0.82
provenance:
    - 'Cluster creation: EKS nodegroup main-20260713055210748600000017 at ~2026-07-13T05:52:10Z; victoria-metrics-k8s-stack deployed with default Kubernetes mixin alerting rules'
---

## Decision

- **why keep:** Newly provisioned EKS cluster deployed the VictoriaMetrics monitoring stack with default Kubernetes mixin alerting rules, but no scrape configuration for EKS control plane components (kube-scheduler, kube-controller-manager, apiserver). On EKS these run on AWS-managed infrastructure and are not exposed as Prometheus scrape targets by default, so the KubeSchedulerDown alert (along with KubeControllerManagerDown and KubeAPIDown) fires from the moment monitoring comes online.
- **confidence:** 82%
- **provenance:** Cluster creation: EKS nodegroup main-20260713055210748600000017 at ~2026-07-13T05:52:10Z; victoria-metrics-k8s-stack deployed with default Kubernetes mixin alerting rules

## Symptom

KubeSchedulerDown — false critical alert on newly provisioned EKS cluster (control plane scrape targets never configured)

Affected resource: VMAlert/victoria-metrics-k8s-stack observability/victoria-metrics-k8s-stack

## Investigate

- EKS nodegroup main-20260713055210748600000017 created at ~05:52 UTC on 2026-07-13 — the timestamp is embedded in the nodegroup name (20260713055210). All metrics first appear at ~06:23 UTC.
- up{job="kube-scheduler"} has NO series at all (not even up=0) across both the 240-minute and 24-hour lookback windows — the target was never configured for scraping, not recently removed.
- up{job="kube-controller-manager"} and up{job="apiserver"} also have NO series — all three control plane component targets are absent.
- ALERTS{alertname=~"KubeSchedulerDown|KubeControllerManagerDown|KubeAPIDown"} all show alertstate=firing simultaneously, yet the cluster is fully functional: pods are Running, Flux Kustomizations reconcile successfully, Kubernetes API calls succeed (pod_status, kube_events, what_changed all return data).
- KubeSchedulerDown alert went pending at 06:22 UTC and firing at 06:32 UTC — immediately after the monitoring stack (victoria-metrics-k8s-stack in observability namespace) came online on the new cluster.
- 30+ scrape jobs are active (kubelet, cilium, core-dns, flux, karpenter, node-exporter, etc.) but none are control plane components — confirming the scrape config never included them.
- No GitOps change to observability caused this: what_changed for observability namespace shows no diff, the observability Kustomization is Ready=True with ReconciliationSucceeded, and the Helm chart's default rules include the Kubernetes mixin alerts that assume control plane targets exist.

## Cause

1. **Newly provisioned EKS cluster deployed the VictoriaMetrics monitoring stack with default Kubernetes mixin alerting rules, but no scrape configuration for EKS control plane components (kube-scheduler, kube-controller-manager, apiserver). On EKS these run on AWS-managed infrastructure and are not exposed as Prometheus scrape targets by default, so the KubeSchedulerDown alert (along with KubeControllerManagerDown and KubeAPIDown) fires from the moment monitoring comes online.** (82%) — change: Cluster creation: EKS nodegroup main-20260713055210748600000017 at ~2026-07-13T05:52:10Z; victoria-metrics-k8s-stack deployed with default Kubernetes mixin alerting rules

## Resolution

- Disable or silence the Kubernetes control-plane component alerts (KubeSchedulerDown, KubeControllerManagerDown, KubeAPIDown, KubeEtcdDown) in the VictoriaMetrics/Prometheus alerting rules for this EKS cluster, since the control plane is AWS-managed and not directly scrapeable. Alternatively, if control plane metrics are desired, configure scraping of the EKS control plane API endpoint (requires network access and proper authentication). (reversible=true)

## Citations

[1] Cluster creation: EKS nodegroup main-20260713055210748600000017 at ~2026-07-13T05:52:10Z; victoria-metrics-k8s-stack deployed with default Kubernetes mixin alerting rules

