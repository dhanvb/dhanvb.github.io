---
title: "Defining Platform Capabilities Before Selecting Kubernetes or OpenShift, CI/CD or GitOps Products"
date: 2026-10-26
permalink: /posts/defining-platform-capabilities-before-selecting-products/
categories:
  - Platform Engineering
  - Architecture
tags:
  - Enterprise Architecture
  - Platform Engineering
  - Kubernetes
  - OpenShift
  - CI/CD
  - GitOps
  - Technology Selection
excerpt: "Why enterprise platform architecture should begin with requirements, capabilities and operational constraints before evaluating Kubernetes, OpenShift, CI/CD or GitOps technologies."
---

## 1. Introduction: Why Product Selection Should Not Be the First Architecture Decision

Imagine an enterprise planning to modernize its application platform.

The organization wants to move toward microservices, establish consistent software delivery practices, improve security and reduce operational complexity.

The technical discussions begin almost immediately:

- Should we use Kubernetes or OpenShift?
- Should the platform run in AWS, Azure, GCP or an on-premises data center?
- Should we adopt GitHub Actions, GitLab CI/CD, Azure Pipelines or Jenkins?
- Should deployments use Argo CD or Flux?
- Should service networking rely on a service mesh?
- Which monitoring and logging products should we deploy?

These are valid questions.

But there is a more fundamental question that should come first:

**What capabilities does the enterprise platform need to provide, and why?**

Without answering that question, technology selection becomes vulnerable to familiarity, vendor influence, industry trends and the personal preferences of individual engineers.

An organization may end up with a sophisticated collection of tools without a coherent explanation of the problems those tools are supposed to solve.

A platform can be technologically advanced and still be architecturally unsuitable for the organization operating it.

This article explores how I approached that problem in my Enterprise Platform Reference Architecture project.

It builds on the earlier design exercise, [Designing a Long-Lived Enterprise Platform for Microservices Workloads]({{ '/posts/designing-a-long-lived-enterprise-platform-for-microservices-workloads/' | relative_url }}).

The focus here is not choosing a winner between Kubernetes and OpenShift or between competing CI/CD products.

It is establishing a defensible method for making those choices.

The supporting design artifacts and automated validation evidence are available through the [Enterprise Platform Reference Architecture project page]({{ '/projects/enterprise-platform-reference-architecture/' | relative_url }}).

---

## 2. The Reference Enterprise Scenario

Architecture decisions require context.

A product that works well for a small development organization may not be the right choice for a regulated enterprise operating many independent application teams.

Likewise, a heavily integrated platform may be appropriate where centralized governance is essential but unnecessarily complex for organizations with limited operational requirements.

For this design exercise, I established a fictional reference enterprise.

The scenario assumes:

- 20 application development teams
- 100 deployable services
- Development, staging and production environments
- Stateless and stateful workloads
- Shared application infrastructure
- Central security and operational governance
- A small central platform engineering team
- A long-term platform lifecycle

The wider reference architecture considers a design horizon of up to approximately ten years.

That does not mean expecting the same products, releases or architectures to remain unchanged for a decade.

Instead, it means defining stable responsibilities while accepting that their implementation will evolve.

The figures in this scenario are assumptions. They provide a common basis for discussing capacity, governance, operational responsibilities and technology evaluation.

They are not measurements from a production enterprise.

A real technology selection would require validation of actual workloads, availability targets, regulatory requirements, hosting constraints, support obligations, engineering skills and budget.

---

## 3. Understanding the Difference Between a Capability and a Product

One of the most important distinctions in enterprise architecture is the separation of what a system must accomplish from how it accomplishes it.

Consider the requirement:

> The platform must support application workload execution, resource allocation, failure recovery and controlled scaling.

This describes a set of capabilities.

It does not specify Kubernetes.

Kubernetes may be an appropriate implementation. OpenShift, a managed container service or another supported runtime may also satisfy the requirements under different conditions.

Likewise:

> Software artifacts must be built, verified, uniquely identifiable and traceable to their source.

This does not automatically require GitHub Actions or GitLab CI/CD.

Those are implementation candidates.

And:

> Authorized deployment state must be reviewable, consistently applied and monitored for drift.

This does not automatically mandate Argo CD or Flux.

The architecture must first determine the desired behavior, responsibilities, controls and evidence.

A useful decision sequence is:

**Business requirement → Architectural capability → Acceptance criteria → Technology alternatives → Verified decision**

Skipping the middle steps risks allowing product features to define the architecture.

This approach does not guarantee implementation independence.

There will always be migration costs, compatibility constraints and integration dependencies.

But it makes those dependencies explicit instead of hiding them inside product decisions.

---

## 4. Decomposing the Enterprise Platform into Capabilities

I organized the reference platform into eight capability domains.

| Capability | What the platform must provide |
|---|---|
| Workload Runtime | Scheduling, execution, scaling and workload recovery |
| Tenancy and Isolation | Resource separation, access boundaries and team isolation |
| Build and Verification | Controlled builds, tests and traceable outputs |
| Artifact Management | Versioned artifacts, integrity and provenance |
| Deployment and Reconciliation | Authorized changes, intended state and drift control |
| Identity, Security and Policy | Authentication, least privilege, secrets and policy enforcement |
| Observability and Reliability | Metrics, logs, traces, reliability signals and investigation |
| Operations and Lifecycle | Maintenance, upgrades, backup, recovery, capacity and auditability |

These domains are not independent islands.

For example, a deployment capability depends on artifact identity, authorization, configuration management and the runtime's deployment interfaces.

Likewise, isolation depends on more than workload placement.

It requires a combination of identity controls, authorization, networking, resource governance and appropriate runtime boundaries.

A namespace alone cannot be assumed to provide complete security isolation.

Another example is observability.

Installing a monitoring system does not automatically create a reliable platform.

The platform must also define meaningful health signals, alert ownership, failure response and service-level expectations.

This capability model makes it easier to ask whether an implementation is complete.

Instead of asking whether Kubernetes is installed, we can ask whether runtime scheduling, failure handling, access governance and lifecycle management satisfy the platform requirements.

The detailed definitions, responsibilities and acceptance criteria are recorded in the repository's capability catalog.

---

## 5. Translating Requirements into Capabilities

A capability catalog is useful, but it is not sufficient on its own.

The next question is why each capability is required.

To address this, I defined 16 reference requirements across four categories:

- Functional requirements
- Non-functional requirements
- Security and governance requirements
- Operational requirements

Each requirement has a stable identifier, priority, related capabilities and verification method.

Consider three examples.

### Example 1: Environment Separation

**Requirement:** Development, staging and production must have separately governable access and change controls.

This requirement influences:

- Tenancy and isolation
- Deployment and reconciliation
- Identity and security

The implementation must support distinct authorization boundaries and prevent unapproved changes to protected environments.

The verification method should examine whether those controls can actually be enforced.

### Example 2: Artifact Traceability

**Requirement:** A deployed software artifact must be traceable to an approved build and release process.

This requires capabilities across:

- Build and verification
- Artifact management
- Security and policy

The architecture must address how software artifacts are identified, retained, verified and deployed.

A successful CI pipeline execution alone is not enough to establish software supply-chain integrity.

### Example 3: Platform Maintainability

**Requirement:** Platform components must support a predictable upgrade and maintenance lifecycle.

This affects platform operations and lifecycle management.

The evaluation must examine:

- Supported release versions
- Upgrade compatibility
- Dependencies
- Recovery procedures
- Operational responsibilities
- Maintenance effort
- Support arrangements

A product that has excellent runtime functionality but cannot be maintained by the organization's engineering team may be a poor architectural choice.

### Why Traceability Matters

I created a requirement-to-capability traceability matrix to connect these relationships.

The matrix helps establish that every documented requirement is associated with relevant capabilities and a verification approach.

It also makes the architecture easier to change.

When a requirement changes, we can identify which capability domains and decisions may be affected.

That is more useful than searching through a large architecture document for every mention of a particular product.

---

## 6. Kubernetes and OpenShift: Evaluating the Right Question

Kubernetes and OpenShift are often compared as though one must be universally better than the other.

That is not a meaningful architectural conclusion without context.

Kubernetes provides an orchestration foundation, while actual enterprise implementations depend on the selected distribution, supporting components, service provider and operating model.

OpenShift builds on Kubernetes and supplies a more integrated platform approach with additional capabilities and conventions.

An enterprise evaluating these approaches must ask questions beyond their feature lists.

### Runtime and Workload Requirements

Can the platform execute the required workloads?

Does it support the needed deployment, storage, scheduling, networking and scaling patterns?

### Security Requirements

Can the organization enforce its mandatory controls?

How are privileges, identities, tenant boundaries and workload policies managed?

### Operational Requirements

Who maintains the control plane?

Who applies security patches?

How are upgrades planned and validated?

What happens when a shared platform component fails?

How much expertise is required to maintain the environment?

### Support and Lifecycle Requirements

Does the implementation have a support model appropriate to the organization's service commitments?

Are product lifecycle policies compatible with enterprise maintenance windows?

### Total Cost

What is the combined cost of infrastructure, software, engineering, support, integration, maintenance and recovery?

A modular platform may provide flexibility and freedom of component choice while requiring substantial engineering effort.

A more integrated platform may reduce some of that effort while introducing different licensing, support, integration or operating considerations.

Neither outcome should be assumed to be superior without evidence.

**The architecture should identify the operating obligations before deciding who or what will fulfill them.**

---

## 7. Why CI/CD and GitOps Should Not Be Compared as Equivalent Products

Another architectural error is treating CI/CD products and GitOps controllers as direct replacements for each other.

They participate in software delivery, but their primary responsibilities differ.

### Continuous Integration

A CI system typically provides capabilities such as:

- Source retrieval
- Software compilation and packaging
- Automated testing
- Security verification
- Build traceability
- Artifact generation and publication

Tools such as GitHub Actions, GitLab CI/CD, Azure Pipelines and Jenkins can implement parts of this capability.

### Deployment Reconciliation

A reconciliation-based delivery system generally focuses on:

- Observing declared desired state
- Applying deployment configuration
- Detecting differences between desired and actual state
- Reconciling changes
- Reporting deployment status

Argo CD and Flux are possible implementations of GitOps reconciliation for Kubernetes-oriented environments.

### How They Can Work Together

A representative delivery model may follow this sequence:

1. A developer submits a source change.
2. An approved CI workflow executes tests and security checks.
3. The build produces an immutable artifact.
4. The artifact is published to an approved repository.
5. An authorized change updates the target environment's deployment declaration.
6. A reconciliation controller applies the desired state.
7. Runtime health and deployment outcomes are observed.

This is one possible architecture.

It is not mandatory for every enterprise.

A controlled pipeline-driven deployment model may also be suitable when it better satisfies organizational requirements.

The important architectural responsibilities are authorization, artifact integrity, repeatable deployment, state visibility and auditability.

GitOps should be selected because its reconciliation and operating model address specific requirements, not because it is fashionable.

---

## 8. Mandatory Constraints Must Come Before Weighted Scoring

Weighted scoring is frequently used to compare technologies.

It provides a structured way to communicate priorities.

But there is a serious weakness if it is used carelessly.

A technology might receive high scores for functionality, cost and usability while failing a mandatory security requirement.

Its overall weighted score might still be higher than competing products.

That should not automatically qualify it for selection.

For this reason, my reference evaluation uses two decision stages.

### Stage 1: Eligibility Gates

Each candidate is examined against mandatory constraints.

Examples include:

- Identity integration
- Administrative access control
- Security policy enforcement
- Audit evidence availability
- Support lifecycle viability
- Hosting restrictions
- Applicable regulatory requirements

Each gate is recorded as PASS, FAIL or UNKNOWN.

A failed mandatory gate prevents selection unless an authorized exception or compensating control is accepted.

An unknown result requires investigation.

### Stage 2: Weighted Evaluation

Only eligible candidates proceed to the weighted comparison.

The reference criteria are:

| Evaluation criterion | Weight |
|---|---:|
| Security and governance | 25% |
| Operational maintainability | 20% |
| Functional capability | 20% |
| Reliability and lifecycle | 15% |
| Total cost of ownership | 10% |
| Organizational skills | 10% |
| **Total** | **100%** |

These weights reflect an illustrative scenario.

They are not a universal weighting that every enterprise should adopt.

An organization with stringent regulatory obligations might assign more importance to security and auditability.

An organization with limited engineering staff might prioritize maintainability and support.

A cost-constrained organization may place greater importance on total cost.

An architect must identify these priorities with stakeholders before relying on the results.

---

## 9. Worked Example: How Priorities Can Reverse a Technology Decision

To test the evaluation method, I created a hypothetical runtime selection exercise.

This is a mathematical and architectural reasoning exercise.

It is **not a verified comparison of actual Kubernetes or OpenShift products**.

The example assumes that two candidate runtime strategies have already satisfied the mandatory eligibility gates.

- **Strategy A:** A modular Kubernetes-based platform
- **Strategy B:** An integrated OpenShift-based platform

The scores are illustrative values assigned to demonstrate how the evaluation responds to changing priorities.

### Initial Evaluation

| Criterion | Weight | Strategy A | Strategy B |
|---|---:|---:|---:|
| Security and governance | 25 | 3 | 4 |
| Operational maintainability | 20 | 3 | 4 |
| Functional capability | 20 | 4 | 4 |
| Reliability and lifecycle | 15 | 3 | 4 |
| Total cost | 10 | 5 | 2 |
| Organizational skills | 10 | 4 | 3 |

The weighted calculation is:

**Weighted Score = Σ(Weight × Score) / 100**

This produces:

| Strategy | Weighted result |
|---|---:|
| Strategy A | 3.50 / 5 |
| Strategy B | 3.70 / 5 |

Under the initial hypothetical scores and weights, Strategy B ranks higher.

But should that automatically settle the decision?

No.

We need to understand how sensitive the outcome is to the organization's priorities.

### Sensitivity Analysis

Consider a different weighting where the organization places substantially more importance on total cost.

The updated weights become:

| Criterion | New weight |
|---|---:|
| Security and governance | 15 |
| Operational maintainability | 15 |
| Functional capability | 15 |
| Reliability and lifecycle | 10 |
| Total cost | 35 |
| Organizational skills | 10 |
| **Total** | **100** |

Without changing the candidate scores, the weighted results become:

| Strategy | Revised result |
|---|---:|
| Strategy A | 3.95 / 5 |
| Strategy B | 3.20 / 5 |

The ranking reverses.

### What This Actually Demonstrates

This exercise does not prove that one platform is cheaper, more secure or more maintainable in a real enterprise.

It demonstrates that technology decisions depend on:

1. The evaluation criteria
2. The importance assigned to each criterion
3. The evidence supporting candidate scores
4. Organizational constraints
5. The assumptions used in the decision

Even a mathematically correct score is not necessarily a defensible architecture decision.

The underlying evidence and decision context matter.

A real evaluation would require validated product capabilities, current support terms, total cost estimates, operational review and, where appropriate, proof-of-concept testing.

---

## 10. Why Total Cost of Ownership Must Be Part of Architecture

One recurring mistake in platform engineering is evaluating technology based only on infrastructure price or software licensing.

That leaves out the actual cost of owning and operating the platform.

Total cost of ownership can include:

- Infrastructure and hosting
- Software subscriptions
- Engineering implementation effort
- Integration and testing
- Operational staffing
- Training
- Security maintenance
- Backup and recovery
- Incident response
- Product support
- Version upgrades
- Migration and eventual replacement

An open-source platform may not require a commercial software license but can still demand substantial operational expertise.

A commercial platform may have higher direct licensing costs while reducing some integration and support responsibilities.

The correct comparison must account for both.

A platform that is inexpensive to install but expensive to maintain over several years may not be the right enterprise investment.

This is particularly important for long-lived platforms, where operational expenses can continue long after the initial implementation has finished.

---

## 11. Architecture Decisions Need Rejected Alternatives

A design document that lists only one chosen technology is missing an important part of architectural reasoning.

It does not explain why the decision was appropriate.

For A02, I considered three general decision approaches.

### Approach 1: Select Familiar Products First

Choose products based on previous engineering experience and construct the architecture around them.

This can reduce the initial learning curve.

However, familiarity cannot establish that a technology meets the organization's requirements.

I rejected this as the general architecture selection method.

### Approach 2: Select an Integrated Platform Suite by Default

Choose a product suite that provides a large portion of the platform's capabilities.

This approach may reduce integration complexity.

However, it can also introduce different dependencies, costs and operating constraints.

I retained integrated platforms as valid candidates, but not as an automatic default.

### Approach 3: Capability-Driven Technology Evaluation

Define requirements and capabilities first.

Apply mandatory constraints.

Evaluate alternatives with documented criteria and evidence.

Consider risks, costs and long-term operations.

This is the selected architectural approach.

The rationale and consequences are recorded in Architecture Decision Record ADR-002.

Importantly, this ADR selects a decision-making method.

It does not prematurely select Kubernetes, OpenShift or a particular software delivery product.

---

## 12. Risks Identified During the Design

A technology evaluation framework also needs to acknowledge how its decisions can fail.

The reference architecture records eight selection-related risks.

Examples include:

**Selecting products before requirements are validated.** This can create unnecessary integration complexity and poor alignment with organizational needs.

**Allowing weighted scores to override mandatory security constraints.** This can produce a mathematically attractive but unacceptable candidate.

**Underestimating operational staffing.** A platform may be technically capable but unsustainable for the team expected to maintain it.

**Ignoring lifecycle costs.** Initial purchasing or installation expenses may not reflect multi-year ownership costs.

**Confusing CI and GitOps responsibilities.** This can create gaps in software delivery authorization and state management.

**Using unsupported scores.** A comparison can appear objective while being based on unverified assumptions.

These risks have proposed mitigations and accountable roles in the repository's risk register.

The risk entries are reference assessments, not measured production incidents.

---

## 13. Validating the Architecture Evidence

Technical architecture documentation should be internally consistent and independently reviewable.

For this milestone, I created a Python validation script using the standard library.

The script checks:

- Unique requirement identifiers
- Requirement-to-capability mapping coverage
- Valid capability references
- Technology scorecard structure
- Unassessed candidate records
- Worked weighted-score calculations
- Sensitivity-analysis results
- Risk register references

The repository contains 16 requirements mapped to eight capabilities, 18 candidate scorecard records and eight architecture risks.

The automated checks passed.

This is useful because even architecture documentation can contain inconsistent identifiers, invalid references and incorrect calculations.

However, the validation boundary must be clear.

The script verifies selected architecture artifacts.

It does not verify that OpenShift, Kubernetes, GitHub Actions, GitLab CI/CD, Argo CD or Flux meets the requirements of an actual organization.

It does not deploy a platform, benchmark workloads, evaluate vendor support agreements or validate operational recovery.

Those activities belong to future implementation and operations evidence.

The complete validation results and supporting artifacts can be examined from the [Enterprise Platform Reference Architecture project]({{ '/projects/enterprise-platform-reference-architecture/' | relative_url }}).

---

## 14. What This Exercise Changed in My Approach

The most important lesson from this exercise is that architecture is not primarily about choosing technologies.

Technology knowledge is necessary, but it is not sufficient.

An engineer may know how to deploy Kubernetes, build a CI pipeline or configure a GitOps controller.

An architect also needs to determine:

- Whether those capabilities are required
- What constraints apply
- How alternative solutions compare
- What dependencies the decision introduces
- Who will operate the solution
- How the solution will be maintained
- What happens when the technology is replaced
- How the decision can be verified

Another lesson is that weighted scorecards must not create false confidence.

Numbers can make a subjective decision look objective.

That is why the evaluation must distinguish between documented facts, stakeholder priorities, hypothetical examples and unresolved questions.

A defensible decision is one whose assumptions and consequences can be examined, challenged and revised.

---

## 15. How This Connects to the Wider Enterprise Platform Architecture

The capability model provides the foundation for subsequent design decisions.

The next area is application onboarding.

Once the platform has identified its capabilities, it must define how application teams consume them.

This includes questions such as:

- What information must an application team provide?
- Who owns application configuration?
- Who is responsible for secrets?
- Who approves production deployment?
- What recovery expectations must be declared?
- How are exceptions handled?
- Where does platform responsibility end and application responsibility begin?

These questions cannot be resolved by installing a Kubernetes distribution or GitOps controller.

They require explicit architectural contracts.

The platform's remaining architecture work will also examine enterprise software delivery, supply-chain security, environment promotion, observability, reliability and lifecycle governance.

The objective is to develop one coherent reference architecture rather than a collection of unrelated technology demonstrations.

---

## 16. Conclusion

An enterprise platform should not be designed by selecting products first and discovering its requirements afterward.

The architecture should begin with the business and operational problems the platform must solve.

Those requirements should be translated into capabilities, acceptance criteria and mandatory constraints.

Only then should implementation technologies be evaluated.

Kubernetes, OpenShift, CI/CD products and GitOps controllers can all be valuable components of an enterprise platform.

Their value depends on how well they satisfy the organization's specific requirements and how sustainably they can be operated.

For this milestone, the outcome is not a selected vendor.

It is a traceable, reviewable and testable method for reaching future technology decisions.

That is the architectural foundation I want to establish before moving into implementation.

---

## Project and Supporting Evidence

**[Explore the Enterprise Platform Reference Architecture Project]({{ '/projects/enterprise-platform-reference-architecture/' | relative_url }})**

The project page explains the overall architecture, completed design milestones and supporting evidence.

The GitHub repository contains the capability catalog, requirements, technology evaluation framework, ADR-002, traceability matrix, risk register and automated validation scripts.

[View the GitHub source repository](https://github.com/dhanvb/enterprise-platform-reference-architecture)