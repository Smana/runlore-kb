---
type: Incident
title: LearningTestDown — CrashLoopBackOff from missing SECRET_KEY env var
description: 'The learning-test deployment in the sre-lab namespace enters CrashLoopBackOff because the required SECRET_KEY environment variable is not set, so the process exits during startup configuration validation.'
resource: sre-lab/learning-test
tags:
    - runlore
    - incident
    - configuration
timestamp: "2026-06-28T13:10:00Z"
resolved: "2026-06-28T13:15:00Z"
fingerprint: seed0learningtest0causex0missingsecretkey00000000000000000000000000
---

## Decision

- **why keep:** learning-test CrashLoopBackOff was caused by a missing required `SECRET_KEY` environment variable; the container exits during startup config validation. Add the env var (from its Secret) to resolve.
- **confidence:** 95% (confirmed resolved)

## Symptom

`learning-test` in `sre-lab` is in CrashLoopBackOff; pods repeatedly exit on startup.

## Cause

The deployment is missing the required `SECRET_KEY` environment variable. On boot the
process validates its configuration, finds `SECRET_KEY` unset, logs a fatal
configuration error, and exits non-zero — driving CrashLoopBackOff.

## Resolution

Set `SECRET_KEY` on the `learning-test` deployment (sourced from its Kubernetes Secret),
then roll the deployment.
