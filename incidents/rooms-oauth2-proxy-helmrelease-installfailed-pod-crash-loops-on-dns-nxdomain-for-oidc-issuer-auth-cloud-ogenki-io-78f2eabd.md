---
type: Incident
title: 'rooms-oauth2-proxy HelmRelease InstallFailed: pod crash-loops on DNS NXDOMAIN for OIDC issuer auth.cloud.ogenki.io'
description: 'The rooms-oauth2-proxy pod (agent-system/rooms-oauth2-proxy-5fb699bbb5-bzdsh) crash-loops with exit 1 because oauth2-proxy cannot resolve its configured OIDC issuer `auth.cloud.ogenki.io` — CoreDNS at 172.20.0.10:53 returns NXDOMAIN (''no such host''). The pod starts, attempts OIDC discovery by fetching https://auth.cloud.ogenki.io/.well-known/openid-configuration, the DNS lookup fails, and the process exits. This CrashLoopBackOff (3 restarts, back-off 40s) prevents the Deployment from becoming Ready, so Helm''s --wait (5m0s timeout) expires and the HelmRelease goes InstallFailed. The OIDC issuer URL is set in the HelmRelease spec values: `oidc-issuer-url: https://auth.cloud.ogenki.io`. The CiliumNetworkPolicy agent-system/rooms-oauth2-proxy has an egress rule `toFQDNs: matchName: auth.cloud.ogenki.io` on port 443, which also depends on DNS resolution working — Cilium FQDN policies learn IPs from DNS responses, so a persistent NXDOMAIN means the policy can never grant egress either.'
resource: agent-system/rooms-oauth2-proxy
tags:
    - runlore
    - incident
    - pod
    - agent-system
timestamp: "2026-11-17T08:21:49Z"
fingerprint: 78f2eabd97a4a96f5675162205b791dd8a8b75093d9d158624369f8d82dc6d5a
confidence: 0.85
origin: runlore
---

## Decision

- **why keep:** The rooms-oauth2-proxy pod (agent-system/rooms-oauth2-proxy-5fb699bbb5-bzdsh) crash-loops with exit 1 because oauth2-proxy cannot resolve its configured OIDC issuer `auth.cloud.ogenki.io` — CoreDNS at 172.20.0.10:53 returns NXDOMAIN ('no such host'). The pod starts, attempts OIDC discovery by fetching https://auth.cloud.ogenki.io/.well-known/openid-configuration, the DNS lookup fails, and the process exits. This CrashLoopBackOff (3 restarts, back-off 40s) prevents the Deployment from becoming Ready, so Helm's --wait (5m0s timeout) expires and the HelmRelease goes InstallFailed. The OIDC issuer URL is set in the HelmRelease spec values: `oidc-issuer-url: https://auth.cloud.ogenki.io`. The CiliumNetworkPolicy agent-system/rooms-oauth2-proxy has an egress rule `toFQDNs: matchName: auth.cloud.ogenki.io` on port 443, which also depends on DNS resolution working — Cilium FQDN policies learn IPs from DNS responses, so a persistent NXDOMAIN means the policy can never grant egress either.
- **confidence:** 85%

## Symptom

rooms-oauth2-proxy HelmRelease InstallFailed: pod crash-loops on DNS NXDOMAIN for OIDC issuer auth.cloud.ogenki.io

Affected resource: Pod agent-system/rooms-oauth2-proxy-5fb699bbb5-bzdsh

## Investigate

- query_logs (container=oauth2-proxy, ns=agent-system): '[2026/11/17 08:16:35] [main.go:73] ERROR: Failed to initialise OAuth2 Proxy: error initialising provider: could not create provider data: error building OIDC ProviderVerifier: could not get verifier builder: error while discovery OIDC configuration: failed to discover OIDC configuration: error performing request: Get "https://auth.cloud.ogenki.io/.well-known/openid-configuration": dial tcp: lookup auth.cloud.ogenki.io on 172.20.0.10:53: no such host' (x7 across 2 pods, first 08:09:14Z → last 08:16:35Z — persistent, not transient)
- pod_status (ns=agent-system): rooms-oauth2-proxy-5fb699bbb5-bzdsh Running ready=0/1 restarts=3 CrashLoopBackOff: back-off 40s restarting failed container=oauth2-proxy [last termination: Error (exit 1)]
- kube_events (ns=agent-system): BackOff (x12) for rooms-oauth2-proxy pod; Readiness probe failed: Get http://100.64.159.200:4180/ready: connect: connection refused (port never opens because container crashes before serving)
- resource_spec HelmRelease agent-system/rooms-oauth2-proxy: spec.values.extraArgs.oidc-issuer-url = https://auth.cloud.ogenki.io; spec.values.extraArgs.provider = oidc; install.remediation.retries = 3; status.installFailures = 2; lastAttemptedReleaseActionDuration = 5m0.4s (matches the Helm timeout)
- resource_spec CiliumNetworkPolicy agent-system/rooms-oauth2-proxy: egress rule 2 = toFQDNs: matchName: auth.cloud.ogenki.io, toPorts 443/TCP — FQDN-based egress that requires DNS resolution to function
- controller_logs (helm-controller, resource=rooms-oauth2-proxy): install action started 08:03:16 → 'release is in a failed state' at 08:08:16 (5m timeout) → uninstall → reinstall 08:08:29 → failed again 08:13:29 → uninstall → third install attempt 08:13:41 still running
- helm-controller logs show the install→timeout→uninstall→reinstall cycle repeating, with 'failed to wait for object to sync in-cache after patching: context deadline exceeded'
- incident_timeline (ns=agent-system): namespace bootstrap at ~08:03 with cascading failures — egress-proxy Unhealthy at 08:03:45, room-broker FailedMount at 08:04:45, AtlasMigration TransientErr at 08:03:14, ExternalSecret UpdateFailed at 08:11:44, cnpg full-recovery FailedScheduling at 08:05:06 (0/6 nodes available: insufficient cpu/memory)
- pod_status (ns=agent-system, app.kubernetes.io/name=egress-proxy): egress-proxy-589459f95d-dc5v6 Running ready=1/1 age=13m — recovered after initial unhealthy period
- query_metrics: up{namespace=agent-system} shows agent-factory pods at 0 (crash-looping), while room-broker and egress-proxy are at 1 (recovered)
- query_logs (container=egress-proxy): 'egress-proxy starting' at 08:03:45, then stack traces from reflector watch errors 08:09:07-08:11:29, then 'egress-proxy ready' — indicates initial instability during bootstrap
- kube_events: egress-proxy readiness probe failures at 08:03:45 (connect: connection refused), room-broker FailedMount at 08:04:45 (secret xplane-rooms-cnpg-cluster-ca not found) — both recovered later
- cloud_resource_health: EKS nodegroup ACTIVE; Karpenter agents-gvisor nodepool had 1 spot instance terminated=1 — node churn during bootstrap may have disrupted networking

## Cause

1. **The rooms-oauth2-proxy pod (agent-system/rooms-oauth2-proxy-5fb699bbb5-bzdsh) crash-loops with exit 1 because oauth2-proxy cannot resolve its configured OIDC issuer `auth.cloud.ogenki.io` — CoreDNS at 172.20.0.10:53 returns NXDOMAIN ('no such host'). The pod starts, attempts OIDC discovery by fetching https://auth.cloud.ogenki.io/.well-known/openid-configuration, the DNS lookup fails, and the process exits. This CrashLoopBackOff (3 restarts, back-off 40s) prevents the Deployment from becoming Ready, so Helm's --wait (5m0s timeout) expires and the HelmRelease goes InstallFailed. The OIDC issuer URL is set in the HelmRelease spec values: `oidc-issuer-url: https://auth.cloud.ogenki.io`. The CiliumNetworkPolicy agent-system/rooms-oauth2-proxy has an egress rule `toFQDNs: matchName: auth.cloud.ogenki.io` on port 443, which also depends on DNS resolution working — Cilium FQDN policies learn IPs from DNS responses, so a persistent NXDOMAIN means the policy can never grant egress either.** (85%)
2. **The DNS resolution failure for auth.cloud.ogenki.io is likely a symptom of a broader cluster bootstrap connectivity issue. The agent-system namespace is being freshly bootstrapped (all pods created ~08:03, ages 13-28m at time of investigation). The egress-proxy pods (which provide outbound internet connectivity for the namespace) were initially unhealthy (readiness probe failures at 08:03:45) and only became Ready around 08:12-08:13. CoreDNS depends on outbound network access to forward queries to upstream resolvers (VPC DNS); if that path was broken during the initial bootstrap window, CoreDNS would return NXDOMAIN for external domains. The oauth2-proxy pod started crashing at 08:09:14 — before egress-proxy recovered — and continued crashing at 08:16:35, possibly due to CoreDNS NXDOMAIN caching or a still-broken upstream path. Multiple other components also failed simultaneously: agent-factory crash-loops (up=0), ExternalSecrets can't get data from provider, AtlasMigration fails (cnpg-cluster not found), and a cnpg full-recovery pod couldn't be scheduled (insufficient CPU/memory).** (25%)

## Resolution

- Investigate why CoreDNS (172.20.0.10) returns NXDOMAIN for auth.cloud.ogenki.io: (1) verify the domain resolves from a node: `kubectl run dns-test --image=busybox --rm -it -- nslookup auth.cloud.ogenki.io`; (2) check CoreDNS upstream forwarding config and whether nodes can reach the VPC DNS resolver (169.254.169.253 on AWS); (3) check if CoreDNS pods have egress to upstream DNS (kube-system NetworkPolicy/CiliumNetworkPolicy). Once DNS resolves, the pod should start successfully on its next retry; if it remains in CrashLoopBackOff, force a reconcile: `flux reconcile hr rooms-oauth2-proxy -n agent-system --force`. (reversible=true)

## Unresolved

- Whether auth.cloud.ogenki.io is a valid, publicly resolvable domain or an internal name that requires special DNS configuration — a human should verify the domain exists in public DNS and is the correct OIDC issuer URL for this environment.
- Whether the DNS resolution failure is persistent (auth.cloud.ogenki.io genuinely unresolvable from this cluster's CoreDNS) or transient (caused by the bootstrap connectivity issues and now self-healing). The pod was still crashing at 08:16:35, but CrashLoopBackOff exponential backoff means it may not have retried since the last failure. A manual pod restart or `flux reconcile hr rooms-oauth2-proxy -n agent-system --force` would test whether DNS has recovered.
- Whether CoreDNS has egress to upstream DNS resolvers — this requires checking kube-system NetworkPolicies/CiliumNetworkPolicies and node security groups, which could not be inspected with the available tools.

