---
title: "Jenkins Pipeline Reliability: Retry + Deterministic Failures + API-Driven Job Provisioning"
date: 2026-03-03
categories: [devops, jenkins, cicd]
tags: [jenkins, pipeline, retry, automation, curl, csurf, devops]
---

## What problem this solves

In real CI/CD systems, tests sometimes fail for reasons unrelated to code changes: timing issues, network flakiness, unstable dependencies, etc.
A common operational pattern is **bounded retries** on the test stage, paired with solid logging so failures remain visible and debuggable.

This project demonstrates:
- A Jenkins pipeline with `retry(3)` on the **Test** stage
- A **deterministic** simulated failure (fails twice, succeeds on the third attempt)
- Automated provisioning: **create/update the Jenkins job via API** (no UI clicking)
- Automated log collection into `pipeline-output.log`

---

## What I built

### Pipeline stages
- **Build**: prints `Building the application ...`
- **Test**: prints `Testing the application ...`, retries up to 3 times, and prints `Simulated test failure` before retrying
- **Deploy**: prints `Deploying the application ...` only after Test eventually passes

### Post-build actions
- Always: `Pipeline Execution Completed.`
- Success: `Pipeline Successful.`
- Failure: `Pipeline Failed.`

---

## Why deterministic failure matters

If you simulate “random failure”, you cannot reliably prove your retry logic works.
Instead, this pipeline writes a counter file (`.test_attempt`) and fails until attempt #3.
That makes the behavior testable and repeatable.

---

## Automation: Jenkins job provisioning via API

This workflow uses:
- Jenkins **crumb issuer** endpoint (CSRF-safe job creation/build triggers)
- Job creation/update using a Pipeline job XML config
- Build trigger via API
- Console log collection via API (saved into `pipeline-output.log`)

This is closer to real CI bootstrapping than manually clicking through the Jenkins UI.

---

## How to run (lab setup)

From the working directory:

```bash
cd /home/ubuntu/code
bash jenkins_setup.sh
bash scripts/run_all.sh
