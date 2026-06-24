---
type: Incident
title: KubeContainerOOMKilled for oom-app pod
description: The container 'hog' in the 'oom-app' pod is being OOMKilled because its memory limit of 100Mi is too low for the workload.
resource: runlore-test/oom-app
tags:
    - runlore
    - incident
fingerprint: a393844afec8e3d1dd728c341b1c425c579246a066b11692e7066fc94de0ae8f
---

## Decision

- **why keep:** The container 'hog' in the 'oom-app' pod is being OOMKilled because its memory limit of 100Mi is too low for the workload.
- **confidence:** 90%

## Symptom

KubeContainerOOMKilled for oom-app pod

## Investigate

- pod_status_response: oom-app-774f997f49-dwlgm Running ready=0/1 ⚠ hog: OOMKilled (exit 137): ⚠ hog: memory limit 100Mi (OOMKilled ⇒ limit likely too low for the workload)

## Cause

1. **The container 'hog' in the 'oom-app' pod is being OOMKilled because its memory limit of 100Mi is too low for the workload.** (90%)

## Resolution

- Increase the memory limit for the 'hog' container in the 'oom-app' deployment. (reversible=false)

