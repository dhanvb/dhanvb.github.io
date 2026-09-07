---

title: "CI/CD Configuration Governance with Ansible Validation Gates"
date: 2026-09-07
layout: single
classes: wide
author_profile: false
comments: true
categories:
  - posts
tags:
  - Ansible
  - CI/CD
  - Configuration Management
  - Platform Engineering
  - DevOps
  - Validation Gates
  - Configuration Governance
  - AI-Assisted Operations
---

## Overview

A deployment pipeline can complete successfully and still deploy the wrong configuration.

The file may have been copied correctly. Ansible may report `changed`. Every automation step may be green.

But none of that proves that the configuration being deployed was actually approved.

That distinction is the focus of this project.

The implementation demonstrates a configuration-governance pattern built around:

* Git-backed candidate configuration
* a separate approved-release policy
* pre-deployment validation
* fail-closed deployment control
* Ansible-based configuration deployment
* post-deployment verification
* SHA-256 integrity checking
* negative-path testing
* operational evidence
* GitHub Actions CI validation
* CODEOWNERS-based ownership
* a foundation for AI-assisted change review

Repository:

[CI/CD Configuration Governance](https://github.com/dhanvb/cicd-configuration-governance)

---

## The Problem: A Green Pipeline Can Still Be Wrong

CI/CD systems are very good at answering questions such as:

* Did the command execute?
* Did the file copy succeed?
* Did the playbook return exit code `0`?
* Did the service restart?
* Did every pipeline stage complete?

Those are useful questions.

They are not the same as:

> Was the configuration we deployed actually the configuration that was approved?

Consider a simple application configuration:

```yaml
version: "7.2"
app_name: demo-app
environment: dev
owner: platform-team
```

A deployment pipeline can successfully copy this file to a server.

But several governance failures are still possible:

* an unapproved version enters the pipeline
* the wrong environment configuration is selected
* somebody changes an endpoint without approval
* a stale file is deployed
* the source configuration changes between review and deployment
* the deployment copies something different from the candidate
* a manual change causes configuration drift
* the release process treats successful automation as proof of approval

The dangerous case is not always a failed deployment.

Sometimes the dangerous case is a successful deployment of the wrong state.

---

## Control Objective

The control objective for this implementation is:

> The deployment stage must not execute unless the candidate configuration passes the pre-deployment governance checks and matches the approved release version.

After deployment, a second control verifies:

> The deployed configuration must match the approved version and the exact candidate file that entered the deployment stage.

This creates two separate boundaries:

1. **Pre-deployment governance**

   * Decide whether the candidate is allowed to proceed.

2. **Post-deployment verification**

   * Prove that the approved candidate was actually delivered.

These controls answer different questions and should not be treated as interchangeable.

---

## Architecture

The final workflow is:

```text
Git-backed candidate configuration
        |
        v
Pre-deployment governance validation
        |
        v
Approved release policy check
        |
        +--------------------+
        |                    |
        | FAIL               | PASS
        v                    v
Deployment blocked      Ansible deployment
Target unchanged              |
                               v
                    Post-deployment verification
                               |
                               v
                    Version + SHA-256 comparison
                               |
                    +----------+----------+
                    |                     |
                    | FAIL                | PASS
                    v                     v
              Deployment rejected   Deployment accepted
                                          |
                                          v
                                   Evidence recorded
```

The most important part of this architecture is where the first rejection happens.

An unapproved candidate is rejected **before the deployment playbook is allowed to modify the target**.

---

## Repository Structure

The repository separates candidate configuration, approval policy, validation, deployment, evidence, and supporting documentation:

```text
.
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│       └── configuration-governance.yml
├── ai-review/
│   ├── config_change_review_prompt.md
│   ├── risk_matrix.md
│   └── sample_ai_review.md
├── ansible/
│   ├── inventory.ini
│   ├── pre_validate.yml
│   ├── deploy.yml
│   └── post_validate.yml
├── config-repo/
│   └── app-config.yml
├── policy/
│   └── approved-release.yml
├── docs/
│   ├── failure-scenarios.md
│   └── production-scenario.md
├── examples/
│   └── ...
├── scripts/
│   ├── run_deployment.sh
│   └── simulate_bad_config.sh
└── architecture.md
```

The separation is deliberate.

The candidate configuration is not itself the approval decision.

---

## 1. Candidate Configuration

The candidate configuration is stored in:

```text
config-repo/app-config.yml
```

Example:

```yaml
version: "7.2"
app_name: demo-app
environment: dev
owner: platform-team
```

This represents the configuration somebody wants to deploy.

It is only a **candidate** until governance validation accepts it.

That terminology matters.

A file being present in Git does not automatically mean it is approved for deployment.

---

## 2. Approved Release Policy

The approved release version is stored separately:

```text
policy/approved-release.yml
```

Example:

```yaml
approved_version: "7.2"
```

This creates a simple configuration baseline.

The candidate says:

```text
I want to deploy version 7.2
```

The policy says:

```text
Version 7.2 is currently approved
```

The pre-deployment gate compares those two states.

This is more useful than allowing the deployment playbook to assume that whatever happens to be present in the configuration repository is automatically approved.

---

## 3. Pre-Deployment Governance Validation

The first important control is implemented in:

```text
ansible/pre_validate.yml
```

The playbook loads both:

* the candidate configuration
* the approved release policy

It first checks that the required fields exist:

```yaml
- name: Validate required configuration fields
  ansible.builtin.assert:
    that:
      - candidate_config.version is defined
      - candidate_config.app_name is defined
      - candidate_config.environment is defined
      - candidate_config.owner is defined
    fail_msg: "Candidate configuration is missing one or more required fields."
```

It then compares the candidate version with the approved release:

```yaml
- name: Validate candidate version against approved release
  ansible.builtin.assert:
    that:
      - (candidate_config.version | string) ==
        (release_policy.approved_version | string)
    fail_msg: >-
      Pre-deployment validation failed.
      Approved version={{ release_policy.approved_version }},
      candidate version={{ candidate_config.version }}.
```

If the two values do not match, Ansible fails immediately.

That failure is not merely informational.

It is used as the control that prevents the deployment stage from executing.

---

## 4. Why the Deployment Does Not Run After Validation Failure

The deployment sequence is controlled by:

```text
scripts/run_deployment.sh
```

The script begins with:

```bash
#!/usr/bin/env bash
set -euo pipefail
```

It then executes the stages sequentially:

```bash
ansible-playbook \
  -i ansible/inventory.ini \
  ansible/pre_validate.yml

ansible-playbook \
  -i ansible/inventory.ini \
  ansible/deploy.yml

ansible-playbook \
  -i ansible/inventory.ini \
  ansible/post_validate.yml
```

Because `set -e` is enabled, a failure in `pre_validate.yml` terminates the script.

Therefore this sequence:

```text
Pre-validation fails
        ↓
Script exits non-zero
        ↓
deploy.yml is never executed
        ↓
Target configuration remains unchanged
```

This is the key fail-closed behavior.

The deployment is not executed and then declared invalid afterward.

The unapproved candidate is stopped before target mutation.

---

## Pipeline vs Deployment Stage

There is an important terminology distinction here.

The CI workflow itself must obviously start in order to perform the governance validation.

Therefore saying:

> The pipeline must not start before validation

is technically incorrect.

A more precise statement is:

> The deployment stage must not start until the governance stage has succeeded.

In a larger production pipeline, this would normally be represented as separate jobs or stages:

```text
Governance Job
      |
      | success
      v
Deployment Job
      |
      v
Verification Job
```

The deployment job would have an explicit dependency on the governance job.

For example, in GitHub Actions this can be modeled using `needs:`.

The important control is not whether CI started.

The important control is whether a deployment-capable stage was permitted to execute.

---

## 5. Deployment of the Approved Candidate

Once pre-deployment validation succeeds, the deployment playbook is allowed to run:

```text
ansible/deploy.yml
```

The playbook creates the target directory and copies the candidate configuration:

```yaml
- name: Deploy approved configuration
  ansible.builtin.copy:
    src: "{{ config_source }}"
    dest: "{{ config_target_file }}"
    mode: "0644"
```

In this repository, the target is intentionally local:

```text
/tmp/cicd-configuration-governance/demo-app/app-config.yml
```

This keeps the repository safe and reproducible as a CI/CD reference implementation.

A real production implementation could replace the localhost inventory with actual managed infrastructure.

The governance pattern remains the same.

---

## 6. Deployment Evidence

After copying the configuration, the deployment playbook calculates a SHA-256 checksum:

```yaml
- name: Calculate deployed configuration checksum
  ansible.builtin.stat:
    path: "{{ config_target_file }}"
    checksum_algorithm: sha256
  register: deployed_file
```

It then records evidence including:

* timestamp
* application
* version
* checksum

This gives the deployment a traceable record instead of relying only on transient console output.

---

## 7. Why Pre-Deployment Validation Is Not Enough

Passing the governance check only proves that the candidate was allowed to proceed.

It does not prove that the correct file reached the destination.

Several things can go wrong after approval:

* the wrong source path may be used
* a deployment task may copy another file
* the target may already contain unexpected data
* automation may mutate the content
* a later step may overwrite the configuration

For that reason, the workflow performs a second validation after deployment.

---

## 8. Post-Deployment Verification

The post-deployment control is implemented in:

```text
ansible/post_validate.yml
```

The playbook reads the file from the target system:

```yaml
- name: Read deployed configuration
  ansible.builtin.slurp:
    src: "{{ config_target_file }}"
  register: deployed_config_raw
```

It parses the YAML:

```yaml
- name: Parse deployed YAML configuration
  ansible.builtin.set_fact:
    deployed_config: "{{ deployed_config_raw.content | b64decode | from_yaml }}"
```

Then it calculates two SHA-256 values:

1. checksum of the candidate configuration
2. checksum of the deployed configuration

The final assertion checks both:

```yaml
- name: Verify deployed configuration
  ansible.builtin.assert:
    that:
      - (deployed_config.version | string) ==
        (release_policy.approved_version | string)
      - deployed_file.stat.checksum ==
        candidate_file.stat.checksum
```

This proves two things:

```text
deployed version = approved version
```

and:

```text
deployed file = candidate file
```

The second comparison matters because version numbers alone cannot prove file integrity.

---

## Approval Integrity vs Deployment Integrity

There are actually two different integrity questions in this workflow.

### Approval integrity

Was the candidate itself approved?

The current implementation answers this using:

```text
candidate version == approved version
```

### Deployment integrity

Was that candidate deployed without modification?

The implementation answers this using:

```text
candidate SHA-256 == deployed SHA-256
```

This distinction is important.

The SHA-256 verification currently proves **deployment integrity**.

It does not, by itself, prove that the complete candidate file was part of the approval decision.

That leads to an important production consideration.

---

## Current Limitation: Version Approval Is Not Full Content Approval

The current approval policy contains:

```yaml
approved_version: "7.2"
```

Suppose somebody changes this:

```yaml
version: "7.2"
environment: dev
```

to:

```yaml
version: "7.2"
environment: production
```

The semantic version has not changed.

The current pre-deployment version gate would therefore still accept the candidate.

Post-deployment checksum verification would correctly prove that the new candidate reached the target, but it would not prove that the changed `environment` field was part of the approval baseline.

For this reference implementation, the semantic-version gate demonstrates the control flow clearly.

For a stronger production control, the approval policy should be bound to immutable candidate identity.

Possible approaches include:

```yaml
approved_version: "7.2"
approved_sha256: "<expected-sha256>"
approved_commit: "<git-commit-sha>"
```

or using:

* a signed release manifest
* a Git tag
* an immutable artifact digest
* a trusted release metadata object
* an external change-management approval record

Then the pre-deployment gate can prove:

```text
candidate identity == approved identity
```

not merely:

```text
candidate version == approved version
```

That is an important difference in a production-grade governance system.

---

## 9. Negative-Path Testing

A governance control is not proven by showing only its successful path.

The repository therefore includes:

```text
scripts/simulate_bad_config.sh
```

The script deliberately tests an unapproved version.

The test performs the following sequence:

```text
Deploy approved 7.2 baseline
        ↓
Calculate target SHA-256
        ↓
Change candidate to 7.3
        ↓
Run normal deployment workflow
        ↓
Pre-deployment gate fails
        ↓
Deployment never executes
        ↓
Calculate target SHA-256 again
        ↓
Compare before and after
```

The test checks three important conditions.

### Condition 1: The pipeline must fail

```bash
if [[ "$STATUS" -eq 0 ]]; then
    echo "ERROR: Unapproved configuration was accepted."
    exit 1
fi
```

### Condition 2: It must fail for the correct reason

```bash
if ! grep -q "Pre-deployment validation failed" "$EVIDENCE_FILE"; then
    echo "ERROR: Pipeline failed, but not at the expected pre-deployment governance gate."
    exit 1
fi
```

This prevents an unrelated failure from being mistaken for successful governance.

A failed DNS lookup, syntax error, or missing dependency is not evidence that the governance gate worked.

### Condition 3: The previously approved target must remain unchanged

The script records the target checksum before and after the rejected attempt:

```bash
if [[ "$BASELINE_SHA" != "$AFTER_SHA" ]]; then
    echo "ERROR: Target configuration changed during rejected deployment."
    exit 1
fi
```

This is one of the strongest parts of the demonstration.

It proves not only:

> the bad deployment failed

but:

> the bad deployment was rejected before it changed the previously approved target state.

---

## 10. Expected Failure

The approved policy contains:

```yaml
approved_version: "7.2"
```

The negative test changes the candidate to:

```yaml
version: "7.3"
```

The resulting Ansible failure is:

```text
Pre-deployment validation failed.
Approved version=7.2,
candidate version=7.3.
```

Execution stops there.

The deployment playbook is never executed for version `7.3`.

That is the behavior the control was designed to prove.

---

## 11. GitHub Actions Validation

The repository also contains:

```text
.github/workflows/configuration-governance.yml
```

The CI workflow performs several checks:

```text
Repository checkout
        ↓
Python setup
        ↓
Ansible installation
        ↓
Shell syntax validation
        ↓
Ansible syntax validation
        ↓
Approved-path test
        ↓
Rejected-path test
        ↓
Evidence upload
```

This is useful because the repository does not depend entirely on somebody manually running the scripts and declaring that they work.

Both the positive and negative paths are exercised automatically.

The positive test proves:

```text
approved candidate
    → validation succeeds
    → deployment runs
    → post-validation succeeds
```

The negative test proves:

```text
unapproved candidate
    → validation fails
    → deployment is blocked
    → existing target remains unchanged
```

A governance mechanism should test both.

---

## 12. Why the Negative Path Is More Important Than It Looks

Teams often test:

> Can we deploy the approved configuration?

They test much less frequently:

> Can the system reliably prevent us from deploying an unapproved configuration?

Both matter.

A control that works only when everybody behaves correctly is not much of a control.

Negative-path testing verifies the enforcement boundary itself.

For configuration governance, useful negative tests can include:

* unapproved version
* missing mandatory field
* wrong environment
* modified approved artifact
* mismatched checksum
* unknown application
* unsigned release metadata
* unauthorized policy modification
* configuration drift after deployment

The current repository starts with the unapproved-version case because the result is easy to observe and verify.

---

## 13. Configuration Ownership with CODEOWNERS

The repository also contains:

```text
.github/CODEOWNERS
```

Ownership is assigned to governance-sensitive paths such as:

```text
/policy/
/ansible/pre_validate.yml
/ansible/post_validate.yml
/.github/workflows/
```

The intention is to make changes to governance logic visible to the appropriate owner.

However, there is an important distinction:

> A CODEOWNERS file identifies reviewers. It does not automatically enforce approval.

For actual enforcement, the repository also needs branch protection or repository rules configured to require the relevant review before merge.

Production governance therefore has two layers:

```text
Repository policy ownership
        +
Branch/ruleset enforcement
```

Without enforcement, CODEOWNERS is useful metadata but not a hard security boundary.

---

## 14. Separate Configuration from Approval Policy

One useful architectural property of this project is the separation between:

```text
config-repo/app-config.yml
```

and:

```text
policy/approved-release.yml
```

If the candidate configuration itself were treated as its own approval record, the process would effectively say:

> Whatever configuration currently exists is approved because it exists.

That is circular.

Separating desired configuration from approval metadata creates an explicit control boundary.

In larger systems, these may live in:

* separate repositories
* protected branches
* release-management systems
* artifact registries
* change-management platforms
* signed release manifests

The repository keeps them in one project for simplicity, but keeps the concepts separate.

---

## 15. Evidence Is Part of the Control

A deployment control is more useful when it produces evidence.

Useful evidence can answer:

* What version was approved?
* What candidate was evaluated?
* When did validation occur?
* What was deployed?
* What checksum reached the target?
* Did post-validation succeed?
* Why was a rejected candidate rejected?
* Was the existing target modified during the failed attempt?

That information becomes useful during:

* incident investigation
* release review
* audit activity
* rollback analysis
* troubleshooting
* postmortems

A green checkmark is a status.

Evidence explains why that status should be trusted.

---

## 16. What the Current Repository Proves

The current reference implementation demonstrates that:

### Approved path

```text
Candidate version 7.2
        ↓
Approved version 7.2
        ↓
Pre-validation PASS
        ↓
Deployment executed
        ↓
Deployed version verified
        ↓
Candidate/deployed SHA-256 verified
        ↓
Deployment accepted
```

### Rejected path

```text
Candidate version 7.3
        ↓
Approved version 7.2
        ↓
Pre-validation FAIL
        ↓
Deployment not executed
        ↓
Previously approved target unchanged
```

Those are concrete behavioral properties, not merely architectural diagrams.

---

## 17. What This Repository Does Not Claim

This project is a sandboxed reference implementation.

It does not attempt to provide a complete enterprise change-management platform.

It currently does not implement:

* cryptographic signing of approval manifests
* external approval-system integration
* immutable artifact repositories
* production secrets management
* multi-environment promotion
* RBAC for multiple release teams
* rollback orchestration
* policy-as-code engines such as OPA
* production host hardening
* deployment concurrency control
* signed Git commits or release attestations
* full configuration-content approval through a baseline digest

Those would be reasonable extensions for a production platform.

The purpose of this repository is to make the core governance boundary observable:

> validate first, block unapproved state before deployment, verify after deployment, and retain evidence.

---

## 18. Production Hardening

A production implementation should extend the pattern further.

### Bind approval to immutable content

Instead of approving only:

```yaml
approved_version: "7.2"
```

use an immutable reference such as:

```yaml
approved_version: "7.2"
approved_sha256: "<sha256>"
approved_commit: "<commit-sha>"
```

### Separate validation and deployment jobs

Represent governance explicitly in CI/CD:

```text
validate
   |
   v
approve
   |
   v
deploy
   |
   v
verify
```

The deployment job should depend on successful governance.

### Protect approval policy

Changes to approval policy should require:

* protected branch
* required reviewers
* code-owner approval
* audit history

Otherwise someone can simply change both the candidate and the approval policy in the same uncontrolled operation.

### Generate evidence during the pipeline

Evidence should be generated from the exact CI/CD execution rather than maintained manually.

### Use immutable dependencies

For higher-assurance workflows, pin:

* automation dependencies
* container images
* external actions
* Ansible versions

to controlled versions or immutable digests.

---

## 19. AI-Assisted Configuration Review

The repository also contains an experimental direction for AI-assisted configuration review.

The purpose of AI in this design is not:

```text
AI decides → AI approves → AI deploys
```

That would collapse the approval boundary into a probabilistic system.

Instead, the proposed role is:

```text
Configuration diff
       ↓
AI-assisted analysis
       ↓
Risk summary
       ↓
Human / policy decision
       ↓
Deterministic governance controls
       ↓
Deployment
```

AI can help summarize:

* what changed
* possible operational impact
* risk level
* affected environments
* validation requirements
* rollback considerations
* missing context
* whether human review is advisable

But the deterministic validation gate should remain authoritative for machine-enforceable policy.

This is an important design principle:

> AI can assist a governance decision without becoming the governance authority.

---

## 20. Example Production Evolution

A more mature implementation could look like this:

```text
Developer Pull Request
        ↓
Configuration diff analysis
        ↓
Schema + policy validation
        ↓
AI-assisted risk summary
        ↓
Human approval when required
        ↓
Signed release manifest created
        ↓
Approved commit / artifact digest recorded
        ↓
Deployment job becomes eligible
        ↓
Pre-deployment immutable digest verification
        ↓
Deployment
        ↓
Post-deployment checksum verification
        ↓
Health validation
        ↓
Evidence stored
```

At that point, the approved baseline is not simply a version number.

It becomes an immutable relationship between:

```text
approval
+
artifact identity
+
deployment evidence
```

---

## Operational Principle

The central lesson from this exercise is straightforward:

> Automation should not only know how to move a change forward. It should know when it is not permitted to move the change forward.

A mature deployment workflow needs both capabilities.

Without the first, nothing gets deployed.

Without the second, anything can.

---

## Conclusion

Successful automation does not automatically mean successful governance.

This project demonstrates three separate controls:

1. **Pre-deployment governance**

   * Reject an unapproved candidate before deployment.

2. **Controlled deployment**

   * Execute deployment only after the governance gate passes.

3. **Post-deployment verification**

   * Prove that the deployed file matches the candidate and approved version.

The negative-path test then proves something equally important:

> When an unapproved version enters the workflow, the pipeline fails at the governance boundary and the previously approved target remains unchanged.

That is a much stronger result than simply showing a successful Ansible deployment.

The repository is available here:

[CI/CD Configuration Governance](https://github.com/dhanvb/cicd-configuration-governance)

