---

title: "CI/CD Configuration Governance with Ansible Validation Gates"
date: 2026-09-07
layout: single
classes: wide
author_profile: false
comments: true
excerpt: "A reference implementation for rejecting unapproved configuration before deployment, verifying deployed state with Ansible and SHA-256 checks, and capturing governance evidence."
categories:
  - projects
tags:
  - Ansible
  - CI/CD
  - Platform Engineering
  - Configuration Management
  - DevOps
  - Configuration Governance

---

## Project Summary

This project demonstrates a CI/CD configuration-governance control pattern in which an unapproved configuration is rejected **before the deployment stage is allowed to modify the target system**.

The workflow uses:

* Git-backed candidate configuration
* a separate approved-release policy
* Ansible pre-deployment validation
* fail-closed deployment sequencing
* Ansible-based configuration deployment
* post-deployment version verification
* SHA-256 candidate/deployed integrity comparison
* negative-path testing
* deployment and validation evidence
* GitHub Actions CI
* CODEOWNERS-based governance ownership
* an experimental foundation for AI-assisted configuration review

Repository:

[GitHub Repository](https://github.com/dhanvb/cicd-configuration-governance)

Full implementation article:

[CI/CD Configuration Governance with Ansible Validation Gates](https://dhanvb.github.io/posts/cicd-configuration-governance-ansible-validation/)

---

## Problem

A CI/CD pipeline can complete successfully while still deploying an unapproved or incorrect configuration.

Technical success answers:

> Did the automation execute successfully?

Configuration governance must answer a different question:

> Was this configuration allowed to be deployed, and did the target receive exactly what the deployment intended?

This project separates those concerns.

---

## Control Flow

```text
Git-backed candidate configuration
        |
        v
Pre-deployment validation
        |
        v
Approved release policy comparison
        |
        +----------------------+
        |                      |
      FAIL                   PASS
        |                      |
        v                      v
Deployment blocked       Ansible deployment
Target unchanged               |
                                v
                     Post-deployment verification
                                |
                                v
                    Version + SHA-256 validation
                                |
                                v
                       Evidence captured
                                |
                                v
                      Deployment accepted
```

---

## What the Project Demonstrates

### 1. Pre-deployment governance

The candidate configuration is validated before deployment.

If its version does not match the approved release policy, execution stops and the deployment playbook is never run.

### 2. Fail-closed behavior

The deployment runner exits immediately when pre-validation fails.

An unapproved configuration therefore cannot continue into the deployment stage.

### 3. Post-deployment verification

After an approved candidate is deployed, Ansible reads the target configuration and verifies:

* deployed version equals approved version
* deployed SHA-256 equals candidate SHA-256

This verifies that the target received the expected candidate file.

### 4. Negative-path testing

The repository intentionally changes the candidate from approved version `7.2` to unapproved version `7.3`.

The test verifies that:

* the governance gate fails
* failure occurs at the expected pre-deployment control
* deployment is not executed
* the previously approved target checksum remains unchanged

### 5. Operational evidence

The workflow records deployment and validation information so that success or failure can be inspected rather than inferred from a green pipeline indicator alone.

### 6. CI validation

GitHub Actions automatically performs:

* shell syntax checks
* Ansible syntax checks
* approved-path testing
* rejected-path testing
* evidence collection

---

## Repository Components

| Component                                        | Purpose                             |
| ------------------------------------------------ | ----------------------------------- |
| `config-repo/app-config.yml`                     | Candidate application configuration |
| `policy/approved-release.yml`                    | Approved release policy             |
| `ansible/pre_validate.yml`                       | Pre-deployment governance gate      |
| `ansible/deploy.yml`                             | Configuration deployment            |
| `ansible/post_validate.yml`                      | Target-state verification           |
| `scripts/run_deployment.sh`                      | Controlled execution sequence       |
| `scripts/simulate_bad_config.sh`                 | Negative-path governance test       |
| `.github/workflows/configuration-governance.yml` | CI validation                       |
| `.github/CODEOWNERS`                             | Governance-sensitive ownership      |
| `examples/`                                      | Execution evidence                  |
| `ai-review/`                                     | AI-assisted review experiment       |

---

## Important Design Distinction

The CI workflow itself must run in order to perform validation.

The control objective is therefore not:

> Do not start CI until configuration is validated.

The correct objective is:

> Do not allow the deployment stage to execute until the configuration-governance stage succeeds.

In a larger production pipeline, governance and deployment would normally be separate jobs with an explicit dependency between them.

---

## Current Scope

The current approval policy validates the semantic configuration version:

```yaml
approved_version: "7.2"
```

Post-deployment SHA-256 verification then proves that the deployed file matches the candidate file.

This demonstrates the core governance and deployment-integrity pattern.

For stronger production approval integrity, the baseline should also contain an immutable candidate identifier such as:

* SHA-256 digest
* Git commit SHA
* signed release manifest
* immutable artifact digest

That would bind approval to the complete configuration rather than only its semantic version.

---

## AI-Assisted Review Direction

The project also explores how AI could assist configuration change review without becoming the approval authority.

AI can help identify:

* configuration differences
* possible production impact
* risk level
* validation requirements
* rollback considerations
* missing context

The final deployment decision should remain controlled by deterministic policy and, where required, human approval.

---

## Key Engineering Principle

A deployment system should not merely answer:

> Can I deploy this configuration?

It should also answer:

> Am I permitted to deploy this configuration?

And after deployment:

> Can I prove that the approved candidate is what reached the target?

This project demonstrates all three boundaries.

