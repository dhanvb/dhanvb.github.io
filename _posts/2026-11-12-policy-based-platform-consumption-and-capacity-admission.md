---
title: "Designing Capacity Admission and Resource Governance for Enterprise Platforms"
date: 2026-11-12
permalink: /posts/policy-based-platform-consumption-and-capacity-admission/
categories:
  - Platform Engineering
  - Architecture
tags:
  - Enterprise Architecture
  - Platform Engineering
  - Kubernetes
  - Capacity Planning
  - Resource Governance
  - Admission Control
  - Python
  - JSON Schema
  - DevOps
excerpt: "Why a valid application resource request is not a capacity reservation, and how I designed a policy-based model for entitlement governance, capacity admission and traceable decisions."
---

## 1. Introduction: A Valid Resource Request Does Not Mean Capacity Is Available

Consider an enterprise running a shared application platform.

A development team submits a request to deploy an application into production.

The request includes:

- Two application replicas
- 250 millicores of CPU requested per replica
- 256 MiB of memory requested per replica
- A defined CPU and memory limit
- An application owner
- A target environment and service tier

The declaration follows the published onboarding contract.

The JSON Schema accepts it.

The application's resource quantities are correctly formatted.

Does that mean the platform should deploy the application?

**No.**

The declaration establishes what the application needs. It does not establish whether the requesting team is entitled to those resources, whether the production environment has sufficient capacity, or whether the request has been authorized.

Even when the application is entitled to the resources and capacity appears available, one more question remains:

**Has that capacity actually been reserved?**

This distinction becomes important when multiple application teams share the same infrastructure.

Two teams might independently request capacity that appears available at the time of evaluation. If the platform evaluates both requests using the same snapshot, both may appear eligible even though the environment cannot satisfy both simultaneously.

A successful capacity check is therefore not the same as a committed allocation.

In the fourth architecture milestone of my [Enterprise Platform Reference Architecture project]({{ '/projects/enterprise-platform-reference-architecture/' | relative_url }}), I investigated how to represent these responsibilities as explicit platform capabilities.

The objective was to design a policy-based consumption model that separates:

**Declared Demand → Entitlement Evaluation → Capacity Eligibility → Authorization → Reservation → Runtime Enforcement → Observed Consumption**

The current deliverable establishes the architecture and implements an offline policy evaluator with automated static tests.

It does not implement the complete production lifecycle.

That boundary is an important part of the engineering design.

---

## 2. Building on the Application Onboarding Contract

The previous architecture milestone, A03, addressed how application teams should describe their platform requirements.

It introduced a declarative onboarding contract with fields for:

- Application identity and revision
- Accountable team and support contacts
- Target environment and service tier
- Workload replicas and resources
- Network exposure
- Data classification and storage
- Health and recovery requirements

The contract provided a common interface between application developers and platform engineering.

It also separated structural validation, policy evaluation, authorization, provisioning and runtime verification.

The complete design is documented in [Designing an Application Onboarding Contract Between Developers and Platform Engineering]({{ '/posts/designing-an-application-onboarding-contract/' | relative_url }}).

A03 answered:

**What must an application team declare before requesting platform resources?**

A04 addresses the next question:

**How should the platform evaluate that declared demand against independently governed entitlements and available capacity?**

The distinction matters because a valid contract can describe a resource request that the platform cannot safely accommodate.

The A04 design therefore reuses the existing A03 JSON Schema rather than defining another application-facing contract.

This preserves a single canonical interface for application onboarding.

---

## 3. The Reference Enterprise Scenario

The reference architecture considers an organization with approximately:

- 20 application development teams
- 100 deployable services
- Development, staging and production environments
- Shared platform infrastructure
- Both stateful and stateless workloads
- Centrally governed security and operations

These numbers are illustrative architecture assumptions, not measured characteristics of a deployed enterprise.

Application teams submit resource requirements through versioned contracts.

The platform team is responsible for providing shared infrastructure, enforcing governance boundaries, managing platform capacity and maintaining operational capabilities.

As application adoption grows, resource allocation becomes an organizational governance problem rather than simply a scheduler configuration problem.

An application may fit within its own resource limits but exceed the total budget allocated to its team.

A team may remain within its budget while the target production environment lacks sufficient headroom.

The platform needs to evaluate both conditions.

This is the problem A04 is designed to address.

---

## 4. Understanding the Different Meanings of Resource Consumption

Before selecting any admission mechanism, I had to distinguish concepts that are frequently treated as interchangeable.

| Concept | Meaning |
|---|---|
| Resource Request | Capacity declared for scheduling and workload planning |
| Resource Limit | Maximum resource usage permitted by the applicable runtime control |
| Team Entitlement | Resource ceiling assigned to an organizational team |
| Environment Headroom | Capacity available after reservations and commitments |
| Capacity Eligibility | Assessment that the declared demand fits applicable constraints |
| Committed Allocation | Capacity accepted into an authoritative reservation record |
| Observed Usage | Resource consumption measured during execution |

These quantities represent different things.

Consider a Kubernetes container with:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

The CPU request is 250 millicores.

The CPU limit is one core.

These values are not equivalent.

A workload can request a smaller amount of CPU for scheduling purposes while being permitted to use more CPU when runtime capacity is available.

Similarly, a memory limit is not a record of observed memory usage.

When two replicas are requested, the total declared CPU request becomes 500 millicores, while the aggregate CPU limit becomes 2000 millicores.

Neither quantity independently proves that the workload has been admitted or reserved.

**The first architectural requirement was therefore to stop treating all resource-related numbers as the same kind of allocation.**

---

## 5. Translating the A03 Contract into Resource Demand

A04 derives the resource-demand model directly from the existing A03 declaration.

The reference production workload uses these values:

```json
{
  "replicas": 2,
  "resources": {
    "cpuRequest": "250m",
    "cpuLimit": "1",
    "memoryRequest": "256Mi",
    "memoryLimit": "1Gi"
  }
}
```

The evaluator calculates aggregate workload requirements.

### CPU Request Calculation

The formula is:

**Total CPU Request = Replica Count × CPU Request Per Replica**

For this workload:

`2 × 250m = 500m`

The platform therefore evaluates a total CPU request of 500 millicores.

The aggregate CPU limit is:

`2 × 1000m = 2000m`

The limit remains separate from the admission demand.

### Memory Request Calculation

Similarly:

**Total Memory Request = Replica Count × Memory Request Per Replica**

For this workload:

`2 × 256Mi = 512Mi`

The aggregate memory limit is:

`2 × 1Gi = 2Gi`

The evaluator normalizes memory values into integer KiB.

The requested memory becomes:

`512 × 1024 = 524288 KiB`

### Persistent Storage

Persistent storage requires a different interpretation.

The existing A03 contract includes `data.storageGiB`, representing the total persistent storage requested for the declared workload.

A04 does not automatically multiply this value by the replica count.

For example, a declared storage value of 10 GiB means a total request of 10 GiB under the current reference contract.

Per-replica storage provisioning would require an explicitly defined contract extension.

The model also does not infer physical storage consumption from logical capacity alone.

Replication factors, backups, snapshots and storage overhead require separate capacity policies.

### Why Exact Resource Normalization Matters

The Python evaluator uses integer quantities and decimal arithmetic.

For CPU, the normalized unit is the millicore.

For memory, the normalized unit is KiB.

A CPU value such as `1.125` can be represented exactly as 1125 millicores.

However, a value such as `1.0001` cores produces a fractional millicore.

The reference evaluator rejects that quantity instead of silently rounding it.

This behavior is deliberate.

Resource governance should not introduce hidden changes to requested capacity through floating-point approximations or undocumented rounding rules.

The implementation is available in the [A04 Python capacity evaluator](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/contracts/consumption/evaluate.py).

---

## 6. Why Capacity Admission Needs Two Independent Constraints

A resource request must satisfy two separate requirements.

### Constraint 1: Team Entitlement

The enterprise may assign a resource budget to each application team.

For example, a team could be permitted to reserve:

- 3000 millicores of CPU
- 4 GiB of memory
- 40 GiB of persistent storage

Assume the team has already committed:

- 1000 millicores of CPU
- 1 GiB of memory
- 5 GiB of storage

The remaining entitlement is:

| Resource | Entitlement | Existing Commitment | Remaining |
|---|---:|---:|---:|
| CPU | 3000m | 1000m | 2000m |
| Memory | 4 GiB | 1 GiB | 3 GiB |
| Storage | 40 GiB | 5 GiB | 35 GiB |

The requested workload must fit within all applicable remaining resource dimensions.

This prevents a team from consuming resources beyond its governed allocation.

However, team entitlement alone is insufficient.

### Constraint 2: Environment Capacity

The production environment has its own capacity constraints.

For example, assume the following synthetic CPU capacity snapshot:

| Capacity Component | CPU |
|---|---:|
| Allocatable capacity | 12000m |
| Platform reservations | 1000m |
| Safety headroom | 1000m |
| Existing workload commitments | 5000m |
| Remaining environment headroom | 5000m |

The environment headroom calculation is:

`12000m - 1000m - 1000m - 5000m = 5000m`

The platform must avoid double counting reservations.

In this reference model, existing workload commitments are separate from platform reservations and safety headroom.

The complete synthetic fixture is documented in the [production capacity snapshot](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/contracts/consumption/examples/production-snapshot.json).

### The Eligibility Rule

For each governed resource dimension:

**Requested Demand ≤ Team Entitlement Headroom**

and

**Requested Demand ≤ Environment Capacity Headroom**

Both conditions must be true.

A request failing either condition is denied capacity eligibility.

The evaluator independently reports the reasons for rejection.

This gives the platform a more useful response than simply returning an unspecified admission failure.

---

## 7. A Worked Example: Eligible Versus Denied

The A04 reference implementation contains two scenarios evaluated against the same synthetic capacity snapshot.

### Scenario A: Eligible Production Workload

The application requests:

- 500 millicores of CPU
- 512 MiB of memory
- 0 GiB of persistent storage

The available headroom is:

| Resource | Requested | Team Headroom | Environment Headroom |
|---|---:|---:|---:|
| CPU | 500m | 2000m | 5000m |
| Memory | 512 MiB | 3072 MiB | 6144 MiB |
| Storage | 0 GiB | 35 GiB | 100 GiB |

Every requested dimension fits both constraints.

The evaluator returns:

```json
{
  "evaluationKind": "advisory",
  "status": "eligible",
  "reasons": []
}
```

This excerpt omits the remaining decision metadata for readability.

The complete decision also includes the application identifier, declaration revision, team identifiers, environment, request digest, capacity snapshot ID, policy version and normalized demand.

The result means the request is eligible against the supplied snapshot.

It does not mean the workload has been approved, reserved or deployed.

### Scenario B: Insufficient CPU Headroom

Now consider a revised test input with two replicas, each requesting three CPU cores.

The aggregate CPU request becomes:

`2 × 3000m = 6000m`

The remaining entitlement is 2000m, while environment headroom is 5000m.

The request exceeds both constraints.

The evaluator returns:

```json
{
  "evaluationKind": "advisory",
  "status": "denied",
  "reasons": [
    "TEAM_CPU_EXCEEDED",
    "ENV_CPU_EXCEEDED"
  ]
}
```

The distinction between the two reason codes is useful.

One identifies an organizational entitlement problem.

The other identifies an infrastructure capacity constraint.

Increasing the team entitlement alone would not resolve the second constraint.

Adding infrastructure capacity alone would not resolve the first.

This makes the outcome explainable and provides a foundation for controlled remediation.

The repository includes the full [eligible decision](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/contracts/consumption/examples/eligible-decision.json) and [denied decision](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/contracts/consumption/examples/denied-decision.json).

---

## 8. Designing the Logical Capacity Admission Workflow

The proposed architecture separates capacity evaluation from the operations that follow it.

![A04 Platform Consumption and Capacity Admission Workflow]({{ '/assets/images/a04-consumption-admission.svg' | relative_url }})

*Figure 1: A04 logical capacity-admission workflow. The diagram distinguishes the implemented offline evaluator from future authentication, authorization, reservation, runtime enforcement and metering integrations.*

The editable diagram source is available in the [technical repository](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/docs/assets/diagrams/src/05-consumption-admission.dot).

The design establishes the following responsibilities.

### Stage 1: Receive the Application Declaration

The platform receives a versioned application request using the canonical A03 contract.

The submitted document is treated as untrusted input.

Its declared team ownership is not accepted as proof of authorization.

### Stage 2: Validate Structure and Cross-Field Policy

The evaluator validates the declaration against the A03 JSON Schema.

It also applies selected policy checks, such as resource request and limit relationships.

Invalid input is rejected.

This stage does not establish whether the requester is authorized to consume the resources.

### Stage 3: Obtain Capacity and Entitlement Data

The intended architecture requires trusted sources of team entitlement and environment capacity.

The current reference implementation uses a manually supplied, versioned JSON snapshot.

The evaluator validates the snapshot's structure and selected consistency relationships.

It does not verify whether the source is authenticated, authoritative, fresh or representative of a live cluster.

### Stage 4: Normalize Workload Demand

The evaluator converts CPU and memory quantities into exact normalized units.

It calculates total workload demand from replica counts and the declared resource requirements.

### Stage 5: Evaluate Capacity Eligibility

The evaluator checks team entitlement headroom and environment headroom.

It returns a deterministic advisory decision.

A valid request that exceeds available headroom is denied with reason codes.

Malformed or inconsistent inputs are rejected rather than assigned an eligibility result.

### Stage 6: Independent Authorization

A future production service must independently verify the requester and applicable approvals.

The decision must remain bound to the exact declaration content and revision.

An application-supplied team identifier cannot establish authorization.

### Stage 7: Atomic Reservation

A future reservation component must revalidate available headroom and commit resource allocations using concurrency-safe operations.

The current evaluator does not implement this stage.

### Stage 8: Runtime Enforcement and Consumption Measurement

After authorization and reservation, a provisioning component may translate approved requirements into runtime-specific controls.

The architecture then requires independent verification of the actual runtime configuration and resource consumption.

These capabilities remain future implementation work.

---

## 9. The Critical Distinction: Eligibility Is Not Reservation

This was the most important architectural issue in A04.

Consider a production environment with 4000 millicores of remaining capacity.

Two application teams submit requests at approximately the same time.

- Team A requests 3000 millicores.
- Team B requests 3000 millicores.

Both evaluations read the same capacity snapshot.

Each evaluator sees 4000 millicores available.

Each request independently satisfies the capacity condition.

Both may therefore receive an `eligible` result.

But the environment cannot satisfy both requests simultaneously while preserving its defined headroom policy.

The combined demand is 6000 millicores.

The environment has only 4000 millicores available.

This is a classic concurrency problem.

### Why the Offline Evaluator Cannot Solve It

The evaluator performs read-only calculations.

It receives a snapshot, evaluates requirements and returns an advisory decision.

It does not maintain an authoritative allocation ledger.

It does not lock shared capacity.

It does not atomically commit reservations.

Therefore, it cannot prevent another request from consuming the same headroom between evaluation and provisioning.

### What a Production Reservation Mechanism Requires

A future implementation must provide:

1. A trusted, authoritative record of committed allocations.
2. An immutable identity for the requested workload revision.
3. A fresh capacity check at the time of commitment.
4. Atomic admission or an equivalent concurrency-safe transaction.
5. Idempotent retry behavior.
6. Reconciliation after partial failures.
7. Controlled resource release after verified runtime changes.

Conceptually, the reservation operation must behave like a transaction.

It must not independently approve two competing allocations against the same remaining resources.

A passing capacity evaluation is useful evidence of policy logic.

It is not evidence that these transaction guarantees exist.

The architecture decision is documented in [ADR-006: Independent Entitlements and Capacity Admission](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/docs/adr/ADR-006-capacity-admission.md).

---

## 10. Why Kubernetes ResourceQuota Alone Is Not Sufficient

Kubernetes already provides mechanisms for managing workload resources.

Why add a separate consumption-governance model?

Because Kubernetes runtime controls and enterprise resource-allocation decisions solve related but different problems.

A namespace ResourceQuota can constrain resource consumption within its configured scope.

LimitRange can define defaults and constraints for relevant resource objects.

Workload requests influence scheduling.

Resource limits constrain runtime consumption.

These are valuable controls.

However, a ResourceQuota object does not independently establish:

- Whether the requesting person belongs to the declared team.
- Whether the team is entitled to a particular enterprise resource budget.
- Whether capacity remains available across multiple clusters or environments.
- Whether a request has received organizational approval.
- Whether a central reservation ledger has atomically committed the allocation.
- Whether committed and observed resource consumption has been reconciled.
- Whether the resulting resource usage has been attributed to an approved financial cost center.

A namespace is also not, by itself, a complete security isolation boundary.

Kubernetes may eventually be selected to implement part of the reference architecture.

But the enterprise-level governance model should not depend entirely on one Kubernetes configuration object.

The chosen architecture therefore defines an independent logical consumption layer.

Runtime quotas can enforce selected outcomes of that governance process.

They do not replace the complete decision process.

---

## 11. Architecture Alternatives Considered

I considered several models before selecting the two-level advisory capacity assessment.

| Alternative | Benefit | Limitation |
|---|---|---|
| First-come resource allocation | Simple operational model | No meaningful team governance; exhaustion risk |
| Static namespace quotas | Native enforcement within Kubernetes | Does not provide the complete enterprise entitlement and reservation model |
| Environment capacity checks only | Prevents some over-allocation decisions | Does not enforce team-specific budgets |
| Team entitlement checks only | Controls organizational allocations | Ignores actual environment constraints |
| Independent entitlement and capacity assessment | Separates governance from available infrastructure | Requires trusted policy data and a future transaction-safe reservation service |

The selected approach evaluates both team entitlement and environment capacity.

It preserves the ability to implement the model using different underlying platform technologies.

The tradeoff is additional governance complexity.

A production implementation must maintain entitlement policies, capacity snapshots, authoritative commitments and reliable reconciliation.

Those requirements cannot be eliminated by choosing a different admission-controller product.

They must be addressed explicitly.

---

## 12. Building Machine-Readable Capacity Contracts

The A04 reference implementation introduces two new JSON Schemas.

### Capacity Snapshot Contract

The capacity snapshot records:

- Snapshot identifier and capture timestamp
- Policy version
- Environment
- Team identifier
- Team entitlement
- Existing team commitments
- Environment allocatable capacity
- Platform reservations
- Safety headroom
- Existing environment commitments

The snapshot is external to the application declaration.

This prevents the applicant from defining its own entitlement inside the onboarding contract.

However, keeping the data in separate documents does not automatically make it trusted.

A production service must obtain the data from an independently governed source and verify access permissions and freshness.

### Advisory Decision Contract

The decision contract records:

- Application identifier and revision
- Declared team and supplied policy team
- Target environment
- Request digest
- Snapshot identity
- Policy version
- Normalized resource demand
- Remaining headroom
- Eligibility status
- Reason codes

The output explicitly identifies itself as an advisory evaluation.

The evaluator binds the decision to a SHA-256 digest of a deterministic JSON serialization.

The serialization uses sorted keys and compact UTF-8 output.

This provides repeatable identity within the reference Python implementation.

It is not presented as a universally interoperable canonicalization standard or an authorization token.

A production design would need to formalize request identity, signing, provenance and trust requirements independently.

The contracts are available in the repository:

- [Capacity Snapshot Schema](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/contracts/consumption/capacity-snapshot.schema.json)
- [Advisory Decision Schema](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/contracts/consumption/decision.schema.json)

---

## 13. Managing Resource Changes and Decommissioning

Initial workload admission is only one part of consumption governance.

Applications change.

A service might increase its replica count, request more memory or reduce its persistent storage requirements.

A production platform needs to distinguish new allocations from changes to existing commitments.

Suppose an application already has a committed CPU allocation of 1000 millicores.

A revised contract requests a total of 1500 millicores.

The proposed additional demand is:

`1500m - 1000m = 500m`

A reservation system must not simply charge or reserve the entire 1500m again.

It must identify the existing application allocation and evaluate the change.

Similarly, reducing the declared resource demand does not necessarily prove that resources have been released.

This is particularly important for persistent storage.

A lower storage requirement in a new contract does not prove that a volume has been safely resized, deleted or reclaimed.

A correct lifecycle must reconcile the committed allocation against the verified runtime state.

The A04 architecture defines these requirements but does not implement allocation updates, storage reclamation or reservation transactions.

Those capabilities are reserved for future implementation and runtime validation.

---

## 14. Resource Accounting: Requests, Commitments and Observed Usage

The second major A04 architecture decision concerns consumption accounting.

An enterprise needs to understand which teams consume shared platform resources.

However, it is misleading to describe declared resource requests as actual consumption.

Consider a service that requests two CPU cores but usually consumes only a fraction of that amount.

Its requested capacity, committed allocation and observed CPU usage describe different conditions.

The architecture therefore separates four records.

### Declared Demand

What the application team requests through its onboarding contract.

### Committed Allocation

What an authorized reservation system has accepted into the authoritative capacity ledger.

### Enforced Configuration

What the execution platform has actually configured and verified.

### Observed Usage

What telemetry records during a defined reporting interval.

These quantities may differ.

A healthy governance model needs to reconcile them without treating missing measurements as zero resource consumption.

### Allocation-Based Showback

An organization may estimate costs using committed resources and allocation duration.

Conceptually:

`Allocated Cost = Committed Quantity × Unit Rate × Duration`

This model can help teams understand the cost of capacity reserved for their workloads.

### Usage-Based Showback

Alternatively, the organization may estimate costs from observed resource consumption.

This requires reliable measurements, reporting intervals, ownership attribution and rate-card rules.

Neither model is automatically equivalent to the infrastructure provider's actual bill.

Shared platform services, network transfer, backup storage, replication overhead and other costs may require separate attribution policies.

A04 documents these architectural distinctions.

It does not implement metering, billing, financial chargeback or an approved rate card.

The decision is recorded in [ADR-007: Separate Commitments, Runtime Configuration and Usage for Showback](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/docs/adr/ADR-007-consumption-accounting.md).

---

## 15. Implementing the Offline Capacity Evaluator

To validate selected architectural rules, I implemented a small Python evaluation component.

The evaluator reads:

1. A valid A03 application declaration.
2. An A04 capacity snapshot.

It then performs structural checks, selected policy validation, resource normalization and headroom comparisons.

A simplified view of the calculation is:

```python
team_headroom = entitlement - team_committed

environment_headroom = (
    allocatable
    - platform_reserved
    - safety_headroom
    - environment_committed
)
```

These are vector operations performed independently for CPU requests, memory requests and storage.

For each dimension, the evaluator checks:

```python
if demand > team_headroom:
    deny_team_entitlement()

if demand > environment_headroom:
    deny_environment_capacity()
```

The actual implementation uses structured resource dictionaries and explicit reason codes.

The full implementation also rejects duplicate JSON keys, validates document schemas, checks team and environment consistency, and verifies selected snapshot invariants.

### Running the Reference Evaluator

From the repository root:

```bash
python -m pip install -r contracts/onboarding/requirements-test.txt

python contracts/consumption/evaluate.py \
  --declaration contracts/examples/valid-production.json \
  --snapshot contracts/consumption/examples/production-snapshot.json
```

For the supplied example, the expected output is an `eligible` advisory decision.

The evaluator uses distinct exit codes:

| Exit Code | Meaning |
|---|---|
| `0` | Valid and capacity-eligible |
| `2` | Rejected or invalid input |
| `3` | Valid request denied by capacity constraints |

This makes the tool suitable for deterministic offline testing.

The implementation does not contact a live cluster, update Kubernetes resources or maintain a reservation database.

Its purpose is to make selected architectural policies executable and testable.

---

## 16. Security and Trust Boundaries

A capacity evaluator can produce mathematically correct results while still being unsafe to integrate with production systems.

This happens when its input data is not independently trusted.

### Applicant-Controlled Team Identity

The A03 declaration contains a team identifier.

The evaluator checks that this value matches the supplied snapshot team.

That comparison is useful for data consistency.

It does not authenticate the requester or prove the team owns the application.

A production implementation must resolve verified identity and organizational entitlement from trusted systems.

### Snapshot Integrity and Freshness

The reference capacity snapshot contains a timestamp and policy version.

Those fields make the input identifiable.

They do not prove that the snapshot was issued by an authorized system or still reflects current capacity.

The current evaluator is not a trusted capacity authority.

### Eligibility and Authorization

An eligible decision must not automatically grant provisioning permissions.

A future production workflow must independently check approvals, identity, target environment permissions and the immutable declaration being authorized.

### Reservation Integrity

Concurrent requests require atomic allocation updates.

Advisory evaluation cannot provide that guarantee.

### Runtime Verification

Even a correctly authorized and committed allocation must be verified after provisioning.

Creating resource objects is not sufficient proof that workload scheduling, isolation and operational requirements have succeeded.

These boundaries are intentional parts of the A04 architecture.

They are documented as outstanding requirements rather than represented as implemented controls.

---

## 17. Testing the Architecture Through Executable Evidence

The A04 repository includes several complementary evidence layers.

### Machine-Readable Contract Validation

JSON Schema Draft 2020-12 validates the capacity snapshot and advisory decision formats.

The schema rejects unsupported fields and invalid structural combinations.

### Unit Testing

The Python unit tests exercise scenarios including:

- Eligible resource requests
- Requests exceeding team entitlement
- Requests exceeding environment capacity
- Memory and storage constraints
- Exact CPU and memory conversion
- Resource request and limit consistency
- Environment and team mismatches
- Malformed and internally inconsistent snapshots
- Duplicate JSON keys
- Deterministic request digests
- Expected decision contract outputs

These tests validate selected properties of the offline evaluator.

They do not prove the safety of production reservation transactions.

### Repository Validation

A separate repository test suite checks whether expected A04 evidence files exist, whether requirements are traceable, and whether architecture indexes contain the relevant references.

### Diagram Verification

The architectural workflow is maintained as editable Graphviz source.

The CI workflow verifies that it renders successfully into SVG.

### GitHub Actions Validation

The A04 static validation workflow executed successfully against commit:

`9d79280b6df8f9b376d760d4d9d0942846be6aeb`

The successful run includes A03 contract regression, A04 evaluator tests, A02/A03/A04 repository tests, Graphviz rendering and the A04 workflow's whitespace check.

**[View A04 GitHub Actions Validation Run #1](https://github.com/dhanvb/enterprise-platform-reference-architecture/actions/runs/43100393538)**

A separate A03 workflow triggered by the same commit failed at its whitespace validation step because newly added Markdown contained trailing spaces. Its contract, architecture and regression checks passed.

The distinction matters: the successful A04 workflow supports the tested static behaviors, while the repository's independent CI workflows were not all green at that revision.

The validation record is maintained in the [A04 Validation Report](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/validation/reports/A04-validation.md).

---

## 18. Requirements Traceability and Architectural Risks

A04 defines twelve architecture requirements, A04-R01 through A04-R12.

They cover:

- A03 schema compatibility
- Exact resource normalization
- Replica-based demand calculations
- Team entitlement evaluation
- Environment capacity evaluation
- Malformed input rejection
- Traceable capacity decisions
- Separation of eligibility and authorization
- Safe allocation updates and idempotency
- Concurrency protection
- Consumption reporting
- Runtime enforcement evidence

The requirements are mapped to planned verification methods and evidence artifacts.

The design also maintains a risk register covering failure modes such as:

| Risk | Required Mitigation |
|---|---|
| Forged team identity | Independently verified identity and entitlement |
| Stale capacity snapshot | Fresh, commit-time capacity verification |
| Double-counted reservations | Authoritative ledger and reconciliation |
| Numeric precision errors | Exact normalization and conversion tests |
| Premature storage release | Verified runtime lifecycle actions |
| Advisory decision used as authorization | Separate permission and approval checks |
| Missing telemetry | Explicit incomplete-data reporting |

These mitigations are not all implemented.

The register distinguishes current static verification from integration and runtime controls that still need engineering work.

This is important for maintaining a credible architecture evidence baseline.

A risk documented in an ADR is not automatically a risk resolved in production.

---

## 19. What the Current Implementation Proves and Does Not Prove

The A04 deliverable contains a coherent set of architecture documents, decisions, contracts, test fixtures, executable policy logic and CI checks.

The current evidence supports selected static properties.

| Capability | Current Evidence |
|---|---|
| Versioned workload contract reuse | Implemented and tested |
| CPU and memory demand normalization | Implemented and tested |
| Team and environment headroom assessment | Implemented and tested |
| Advisory decision schema | Implemented and tested |
| Selected malformed-input rejection | Implemented and tested |
| Architecture traceability | Static repository checks |
| Authenticated entitlements | Architecture only |
| Atomic reservations | Architecture only |
| Concurrency-safe allocation updates | Architecture only |
| Kubernetes runtime enforcement | Not implemented |
| Live consumption telemetry | Not implemented |
| Financial showback | Not implemented |

This separation prevents the project from overstating its technical maturity.

A static policy evaluator is not a production admission controller.

A successful CI workflow does not prove infrastructure capacity exists.

A capacity snapshot is not an authoritative reservation ledger.

A documented resource policy does not prove that Kubernetes enforces it.

The actual production behavior of these mechanisms would require integration and runtime evidence.

---

## 20. Engineering Lessons from A04

This milestone reinforced several design principles.

### 1. Define the Capacity Model Before Selecting Admission Tools

Kubernetes admission controls, resource quotas and schedulers may all be useful implementation mechanisms.

But the architecture first needs to define what is being governed, which data is authoritative and what qualifies as a successful decision.

Otherwise, tool selection risks concealing missing governance requirements.

### 2. Entitlement Is Different from Availability

Organizational permission to consume resources does not guarantee that those resources are physically available.

Capacity admission must evaluate both constraints.

### 3. Eligibility Is Different from Commitment

A read-only assessment cannot prevent concurrent resource allocation conflicts.

Production resource reservation requires transaction-safe state changes.

### 4. Resource Requests Are Not Observed Consumption

Requested capacity, enforced limits, committed allocation and measured usage have different meanings.

They should not be combined into one undifferentiated resource-consumption figure.

### 5. Architecture Evidence Must Preserve Its Limits

A passing unit test is evidence that a specific rule behaves as expected under tested conditions.

It is not evidence that unimplemented integrations, security boundaries or operational guarantees are functioning.

Making these limits explicit improves the usefulness of the architecture.

---

## 21. What Comes Next?

A04 establishes the consumption-governance design.

Future implementation work will need to validate the controls that static evaluation cannot prove.

Those include:

1. Integration with authenticated organizational identity and entitlement sources.
2. Collection and verification of live environment capacity data.
3. An authoritative resource-allocation ledger.
4. Atomic capacity reservation and safe concurrent admission.
5. Revision-aware allocation updates and lifecycle release.
6. Translation of committed allocations into runtime enforcement controls.
7. Reconciliation between committed and enforced resources.
8. Measured consumption and transparent cost attribution.
9. Operational verification during capacity pressure and partial failure.

These are implementation and operational milestones, not features claimed by the current A04 demonstrator.

The wider Enterprise Platform Reference Architecture project will also continue into software delivery architecture, supply-chain security, GitOps promotion, observability and long-term platform operations.

The goal is to connect these capabilities through consistent contracts, decisions, responsibilities and evidence rather than designing them as unrelated tooling exercises.

---

## 22. Conclusion

An enterprise platform needs more than the ability to accept application resource declarations.

It must be able to determine whether those requests are permitted, whether capacity is available, whether an allocation has been committed and whether the runtime actually enforces the accepted configuration.

A04 addresses the architectural foundation of that problem.

It introduces an independently governed entitlement and capacity-evaluation model, a machine-readable decision contract, an offline policy evaluator and automated static validation.

The most important principle is:

**A valid resource request is not an entitlement. An eligible request is not a reservation. A reservation is not proof of runtime enforcement.**

These distinctions are essential when designing shared enterprise platforms that must remain secure, accountable and operable as application demand grows.

The current result is a documented and statically tested reference design.

The next challenge is to demonstrate how its guarantees can be implemented and verified under real concurrency, capacity pressure and operational failure conditions.

---

## Project and Supporting Engineering Evidence

**[Enterprise Platform Reference Architecture: Project Overview]({{ '/projects/enterprise-platform-reference-architecture/' | relative_url }})**

The project page connects A04 with the earlier capability design, technology evaluation and application onboarding milestones.

The complete technical evidence is maintained in GitHub:

- [A04 Architecture Specification](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/docs/architecture/08-platform-consumption-model.md)
- [Platform Consumption High-Level Design](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/docs/hld/HLD-003-platform-consumption.md)
- [ADR-006: Capacity Admission](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/docs/adr/ADR-006-capacity-admission.md)
- [ADR-007: Consumption Accounting](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/docs/adr/ADR-007-consumption-accounting.md)
- [Capacity Snapshot Schema](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/contracts/consumption/capacity-snapshot.schema.json)
- [Advisory Decision Schema](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/contracts/consumption/decision.schema.json)
- [Python Capacity Evaluator](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/contracts/consumption/evaluate.py)
- [Capacity Evaluation Tests](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/contracts/consumption/test_consumption.py)
- [Requirements Traceability](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/docs/decisions/consumption-traceability.csv)
- [Architectural Risk Register](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/docs/decisions/consumption-risk-register.csv)
- [A04 Validation Report](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/validation/reports/A04-validation.md)
- [A04 Evidence Index](https://github.com/dhanvb/enterprise-platform-reference-architecture/blob/main/evidence/A04.md)
- [Successful A04 GitHub Actions Run](https://github.com/dhanvb/enterprise-platform-reference-architecture/actions/runs/43100393538)

**[Explore the Complete Enterprise Platform Reference Architecture Repository](https://github.com/dhanvb/enterprise-platform-reference-architecture)**