---
title: "Jenkins Retry Pipeline: Deterministic Retries + API-Driven Job Provisioning"
date: 2026-03-05
categories: [DevOps, Jenkins, CI/CD]
tags: [jenkins, pipeline, retry, curl, automation, api, csrf]
---

## Overview

This scenario demonstrates a practical Jenkins reliability pattern: **retrying a flaky test stage** in a controlled way, while also provisioning and triggering the pipeline job **entirely via CLI and Jenkins API** (no UI clicking).

What this covers:

- Jenkins installation on Ubuntu (scripted)
- Pipeline with stages: **Build → Test (retry 3x) → Deploy**
- Deterministic simulated failure to prove `retry(3)` works (fails twice, succeeds on the third attempt)
- Post-build status logging (`always`, `success`, `failure`)
- Job creation/update via Jenkins API with **CSRF crumb** handling
- Console log export to `pipeline-output.log`

---

## Why this matters in real CI/CD

Retries are not a permanent fix, but they are a valid operational tool when:

- tests occasionally fail due to transient dependency/network issues
- you need bounded resilience while you investigate flakiness
- you want predictable behavior and consistent post-build signals

The key is to keep retries **bounded**, **observable**, and **auditable**.

---

## Environment

- Ubuntu machine
- Working directory: `/home/ubuntu/code`
- Jenkins installed via provided script: `jenkins_setup.sh`
- After installation, admin password stored at: `/home/ubuntu/code/jenkins_admin_password`
- Jenkins URL: `http://localhost:8080`

> If a Jenkins installation fails due to a GPG key error, follow the troubleshooting steps in the lab `README.md`.

---

## Pipeline Requirements

### Stages
1. **Build**
   - Prints: `Building the application ...`
2. **Test**
   - Prints: `Testing the application ...`
   - Retries the stage up to **3 times** on failure
   - Prints error message **before** retrying: `Simulated test failure`
3. **Deploy**
   - Prints: `Deploying the application ...`
   - Runs only if the pipeline reaches this stage (tests pass eventually)

### Post-build actions
- Always: `Pipeline Execution Completed.`
- Success: `Pipeline Successful.`
- Failure: `Pipeline Failed.`

---

## Jenkinsfile (Pipeline logic)

The pipeline uses `retry(3)` to rerun the Test stage if it fails.  
To make retry behavior deterministic (not random), it writes a `.test_attempt` file in the workspace and fails until attempt #3.

```groovy
pipeline {
  agent any

  stages {
    stage('Build') {
      steps {
        echo 'Building the application ... '
      }
    }

    stage('Test') {
      steps {
        retry(3) {
          echo 'Testing the application ... '
          script {
            def f = '.test_attempt'
            def n = 0
            if (fileExists(f)) {
              n = readFile(f).trim().toInteger()
            }
            n = n + 1
            writeFile(file: f, text: "${n}\n")

            if (n < 3) {
              echo 'Simulated test failure'
              error('Simulated test failure')
            }
          }
        }
      }
    }

    stage('Deploy') {
      steps {
        echo 'Deploying the application ... '
      }
    }
  }

  post {
    always  { echo 'Pipeline Execution Completed.' }
    success { echo 'Pipeline Successful.' }
    failure { echo 'Pipeline Failed.' }
  }
}
```

Full Github Repo - `https://github.com/dhanvb/devops-scenarios/tree/main/scenarios/jenkins/001-retry-pipeline`
