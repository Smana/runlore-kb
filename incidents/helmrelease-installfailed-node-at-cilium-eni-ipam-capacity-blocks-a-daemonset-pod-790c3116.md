---
type: Incident
title: 'HelmRelease InstallFailed: a node at Cilium ENI IPAM capacity blocks a DaemonSet pod'
description: 'A single node has exhausted its Cilium per-node IPAM capacity, so a DaemonSet pod scheduled there cannot get a pod IP and stays ContainerCreating with `FailedCreatePodSandBox ... cilium-cni failed (add): unable to allocate IP via local cilium agent: no IPs currently available on the node`. Because a DaemonSet is only complete when it has a pod on every node, it never leaves status InProgress, so Helm''s --wait times out and the owning HelmRelease goes InstallFailed — in a namespace that looks unrelated to the real fault. First seen as observability/victoria-logs blocked by node ip-10-0-3-147 at 42/42 IPv4.'
resource: observability/victoria-logs
tags:
    - runlore
    - incident
    - helmrelease
    - observability
    - cilium
    - ipam
    - eni
    - daemonset
    - node
    - aws
timestamp: "2026-08-02T10:02:27Z"
last_validated: "2026-08-02"
fingerprint: 790c311646f2f1ad937e1024332cefc503c490c28211ddb66d67ae098ba0396f
confidence: 1.0
provenance:
    - No GitOps change caused this — what_changed shows no recent diff for observability; the node simply reached its Cilium per-node IPAM ceiling organically as pods accumulated.
---

## Decision

- **why keep:** A node at its Cilium per-node IPAM ceiling starves a DaemonSet pod of an IP, and the failure surfaces as a HelmRelease `InstallFailed` in whatever namespace happens to own that DaemonSet. The reported symptom points at the chart; the fault is a node three layers away. That misdirection is the reusable part.
- **confidence:** 100% (reproduced and fixed end to end on 2026-08-02)

## Symptom

A HelmRelease that ships a DaemonSet never completes:

```
Helm install failed for release observability/victoria-logs with chart
victoria-logs-single@0.13.9: timeout waiting for:
[DaemonSet/observability/victoria-logs-vector status: 'InProgress']
```

Exactly one of the DaemonSet's pods is stuck `ContainerCreating` (`READY 8/9`),
and it is always on the same node. Every other pod of the same DaemonSet is
`1/1 Running`. Nothing is wrong with the chart.

## Investigate

- `pod_status`: the stuck pod is `Pending`/`ContainerCreating`, `ready=0/1`, pinned to one node
- `kube_events`: `FailedCreatePodSandBox (x201)`: `failed to setup network for sandbox ... plugin type="cilium-cni" failed (add): unable to allocate IP via local cilium agent: [POST /ipam][502] postIpamFailure "no IPs currently available on the node, allocation will be retried once Cilium Operator allocates more IPs"`
- `query_metrics`: `cilium_ipam_capacity{node="…"}` equals `cilium_ip_addresses{node="…",family="ipv4"}` → 100% utilisation (42 = 42 in the observed case)
- `query_metrics`: `cilium_operator_ipam_nodes{category="at-capacity"} = 1` against `{category="total"} = 9` → exactly one node exhausted
- `query_metrics_range`: the node's `cilium_ip_addresses` is **flat** at the ceiling across the whole window — it was already saturated before the incident, so this is not a leak or a spike
- All other pods of the DaemonSet are `Running 1/1` on other nodes — the failure is isolated to one node's IP pool

Confirm directly on the node, which is faster and more decisive than the metrics:

```bash
# Which node is the stuck pod on
kubectl get pod -n <ns> <pod> -o jsonpath='{.spec.nodeName}{"\n"}'

# Ask that node's Cilium agent — "42/42 allocated" is the whole diagnosis
CP=$(kubectl get pods -n kube-system -l k8s-app=cilium \
       --field-selector spec.nodeName=<node> -o name | head -1)
kubectl exec -n kube-system ${CP#pod/} -c cilium-agent -- \
  cilium-dbg status --all-addresses | grep -A5 IPAM

# How packed is the node really?
kubectl get pods -A --field-selector spec.nodeName=<node> --no-headers | wc -l
```

## Cause

1. **The node has exhausted its Cilium per-node IPAM capacity, so the DaemonSet pod cannot be allocated a pod IP.** (100%)

   The chain: node at `cilium_ip_addresses == cilium_ipam_capacity` → CNI ADD returns
   502 `no IPs currently available on the node` → the pod never gets a sandbox → the
   DaemonSet cannot reach a pod-per-node and stays `InProgress` → Helm `--wait` times
   out → `HelmRelease InstallFailed`. No GitOps change is involved; the node reached
   the ceiling organically as pods accumulated.

   **The ceiling is a property of the instance, not a tunable pool.** This cluster runs
   Cilium `ipam: eni`, where per-node capacity is bounded by the instance type's ENI
   limits (ENIs × IPs-per-ENI), so there is no pool size to raise on that node. The
   observed node was carrying **46 pods** — it was legitimately full, not leaking.

2. ~~The HelmRelease has exhausted its install remediation retries and needs `flux reconcile --reset`.~~
   **Refuted on 2026-08-02.** helm-controller did log `terminal error: exceeded maximum
   retries: cannot remediate failed release`, which is what motivated this hypothesis —
   but once the IP was freed, a plain `flux reconcile hr victoria-logs -n observability
   --force` applied revision `0.13.9` and went Ready. Do not reach for `--reset`, and
   especially not `helm uninstall`, until plain `--force` has actually failed.

## Resolution

Verified end to end on 2026-08-02.

1. **Free one IP on the affected node.** Delete a pod that has a replica elsewhere — it
   reschedules onto another node and the IP is released:

   ```bash
   kubectl delete pod -n <ns> <replicated-pod-on-that-node>
   ```

   Pick a pod belonging to a Deployment/ReplicaSet, never the DaemonSet pod you are
   trying to start (it would just be recreated on the same node). In the observed case
   both `metrics-server` replicas were on the exhausted node, so deleting one freed an
   IP *and* fixed an HA anti-affinity gap.

   **If the node is cordoned, the freed IP stays free** — the evicted pod cannot come
   back. And note that **cordoning does not relieve this on its own**: DaemonSet pods
   tolerate `node.kubernetes.io/unschedulable`, so they are still placed on cordoned
   nodes. A node that is both full and cordoned is a standing deadlock for any *new*
   DaemonSet until an IP is freed or the node is drained and removed.

   The stuck pod picks up the freed IP within seconds — no restart needed.

2. **Clear the HelmRelease with a plain force-reconcile:**

   ```bash
   flux reconcile hr <name> -n <ns> --force
   ```

   Escalate to `--reset` (see
   [helmrelease-terminal-failed-exhausted-retries](../helmrelease-terminal-failed-exhausted-retries.md))
   only if that actually fails.

### If it keeps happening

One node at capacity is a scheduling-density problem, not a Cilium bug. The levers, in
order of preference: spread the load (anti-affinity / topology spread on the fat
Deployments — the observed node held both `metrics-server` replicas plus most Flux,
Crossplane and Tailscale singletons); use larger instance types for nodes that attract
control-plane-ish workloads; or move off `ipam: eni` to prefix delegation, which raises
the per-node ceiling substantially. Watch
`cilium_operator_ipam_nodes{category="at-capacity"} > 0` to catch it before a DaemonSet
does.

## Unresolved

- (none — both prior open questions were answered on 2026-08-02. The 42-IP ceiling is
  the instance's ENI limit under `ipam: eni`, not a configured pool that should be
  raised; and there were no stale/leaked IPs — the node genuinely hosted 46 pods.)
