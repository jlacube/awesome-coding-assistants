# Review Spec Completeness & Contract-Aware Review -- Specification

> **Source brief**: `.sdd/ideas/002-sdd-pipeline-v2-universal-skill-architecture.md`
> **Feature branch**: `005-review-spec-completeness`
> **Status**: Draft
> **Version**: 1.0

---

## 1. Overview

Add a new `review-spec-completeness` skill to the Review Coordinator's skill catalog and expand the existing `review-spec` skill to validate code against formal contract files (not just prose spec text). The review-spec-completeness skill serves as a pre-planning validation gate, ensuring specifications are deep enough for autonomous implementation before the Planner decomposes them. The review-spec expansion enables the Reviewer to check that implemented code matches contract definitions (interfaces, data schemas, API contracts, state machines, error catalogs) field-by-field rather than relying on prose interpretation.

---

## 2. Goals & Success Criteria

- **SC-001**: The review-spec-completeness skill catches shallow or incomplete specs before the Planner runs, preventing downstream WP quality issues. Verified by: specs that fail the completeness check produce specific, actionable findings.
- **SC-002**: The review-spec skill validates implemented code against formal contract files, catching field name mismatches, type mismatches, missing error codes, and incorrect state transitions. Verified by: every contract definition has a corresponding check in the implementation.
- **SC-003**: Both skills integrate into the existing Review Coordinator's dynamic discovery mechanism (glob pattern `review-*/SKILL.md`). Verified by: no Review Coordinator changes needed.

---

## 3. Users & Roles

- **Planner Agent (gated consumer)**: Cannot proceed until review-spec-completeness passes. Uses completeness findings to identify spec gaps for auto-loop.
- **Coder Agent (reviewed subject)**: Code is reviewed against contracts by the expanded review-spec skill.
- **Review Coordinator (dispatcher)**: Discovers and dispatches both skills as part of the review skill catalog.
- **Spec Architect (gap resolver)**: Receives completeness findings when the spec needs revision.

---

## 4. Functional Requirements

### 4.1 review-spec-completeness Skill

#### 4.1.1 Purpose and Invocation

- **FR-001**: The review-spec-completeness skill SHALL validate that a specification is implementation-complete before planning begins. It is dispatched by the Review Coordinator or directly by the Planner coordinator during its completeness pre-check.

- **FR-002**: The skill SHALL read the spec file and its companion artifacts directory as inputs.

#### 4.1.2 Completeness Checks

- **FR-003**: The skill SHALL check that every FR uses SHALL or SHALL NOT obligation language. Any FR containing "should", "could", "might", "may", or "can" (as obligation, not permission) SHALL be flagged as FINDING with severity HIGH.

- **FR-004**: The skill SHALL check that every FR has defined error behavior (what happens when the happy path fails). FRs without error paths SHALL be flagged with severity HIGH.

- **FR-005**: The skill SHALL check that every entity in the data model (Section 7) has:
  1. All fields with explicit types (no untyped fields)
  2. Nullability declared for every field
  3. Constraints (required, unique, max length, format, min/max) for every field
  4. Validation rules beyond type constraints
  5. Default values (or explicit "no default")
  - Missing items SHALL be flagged with severity HIGH.

- **FR-006**: The skill SHALL check that every API endpoint (Section 8) has:
  1. All applicable HTTP error codes (400, 401, 403, 404, 409, 422, 500) with meanings and response bodies
  2. Request schema with all fields typed
  3. Response schema with all fields typed
  4. Auth requirements stated
  - Missing items SHALL be flagged with severity HIGH.

- **FR-007**: The skill SHALL check that every entity with a status/state field has:
  1. All valid states listed as an enum
  2. All valid transitions defined (from-state, to-state)
  3. Guards/conditions on each transition
  4. Side effects per transition
  - Missing items SHALL be flagged with severity MEDIUM.

- **FR-008**: The skill SHALL check that the traceability matrix (Section 16) has no empty cells:
  1. Every FR maps to at least one US
  2. Every US maps to at least one acceptance scenario
  3. Every acceptance scenario maps to at least one test type
  4. Every test type maps to a test section reference
  - Empty cells SHALL be flagged with severity HIGH.

- **FR-009**: The skill SHALL check that every external integration (Section 9.5) has:
  1. Timeout value
  2. Retry strategy
  3. Fallback behavior
  4. Circuit breaker threshold (if applicable)
  - Missing items SHALL be flagged with severity MEDIUM.

- **FR-010**: The skill SHALL check for ambiguous language. Any occurrence of "appropriate", "reasonable", "as needed", "etc.", "similar", "relevant" in FR or NFR text SHALL be flagged with severity MEDIUM.

- **FR-011**: The skill SHALL check that companion artifact files (`.sdd/specs/artifacts/`) exist and are consistent with the prose spec:
  1. Every entity in Section 7 has a corresponding type definition in `data-models.<ext>`
  2. Every API endpoint in Section 8 has corresponding request/response types in `api-contracts.<ext>`
  3. Every error code in Section 4 has a corresponding entry in `error-catalog.<ext>`
  4. Field names and types match between prose and artifacts
  - Missing or inconsistent artifacts SHALL be flagged with severity HIGH.

- **FR-012**: The skill SHALL check security requirements (Section 10.2):
  1. Per-component security requirements exist (not just a global statement)
  2. OWASP mitigation references are present
  3. Data sensitivity classification per entity exists
  - Missing items SHALL be flagged with severity MEDIUM.

#### 4.1.3 Output Format

- **FR-013**: The skill SHALL produce findings in the standard review finding format:
  ```
  ### Finding: SPEC-COMP-XXX
  - **Severity**: HIGH | MEDIUM | LOW
  - **Category**: obligation-language | error-behavior | data-model | api-contract | state-machine | traceability | integration | ambiguity | artifact-consistency | security
  - **Location**: Section N, FR-XXX or entity name
  - **Issue**: Description of what is missing or incorrect
  - **Recommendation**: Specific action to fix
  ```

- **FR-014**: The skill SHALL produce a verdict:
  - **PASS**: Zero HIGH findings. The spec is implementation-ready.
  - **FAIL**: One or more HIGH findings. The spec needs revision before planning.
  - Along with finding counts by severity.

#### Implementation Contract -- review-spec-completeness

**Inputs**:
- Spec file path (string).
- Companion artifacts directory (string).

**Outputs**:
- Findings list (structured, per FR-013 format).
- Verdict: PASS or FAIL.
- Finding counts: high, medium, low.

**Error behaviors**:
- Spec file not found: halt, report error.
- Artifacts directory not found: flag as HIGH finding (artifacts missing), do not halt.

---

### 4.2 review-spec Skill Expansion (Contract-Aware Review)

#### 4.2.1 Existing Behavior Preserved

- **FR-015**: The existing review-spec skill behavior SHALL be preserved: validate that implementation code adheres to the prose spec's FRs, user stories, and acceptance scenarios.

#### 4.2.2 Contract-Aware Checks (New)

- **FR-016**: The skill SHALL additionally validate implementation code against contract files in `.sdd/plans/contracts/<WP-slug>/`:

  1. **Interface contract check**: Every function/method signature in the implementation SHALL match the corresponding signature in `interfaces.<ext>` (parameter names, parameter types, return type).
  2. **Data schema check**: Every entity/model class in the implementation SHALL match the corresponding definition in `data-schemas.<ext>` (field names, field types, constraints, defaults).
  3. **API contract check**: Every API endpoint in the implementation SHALL match `api-contracts.<ext>` (method, path, request schema, response schema, error responses).
  4. **State machine check**: Every state transition in the implementation SHALL match `state-machines.<ext>` (valid states, valid transitions, guards).
  5. **Error catalog check**: Every error code/message in the implementation SHALL match `error-catalog.<ext>` (code value, HTTP status, message template).

- **FR-017**: For each contract check, the skill SHALL compare token-by-token:
  1. Function/method names: exact match
  2. Parameter names: exact match
  3. Type annotations: exact match (including generics, nullability)
  4. Field names: exact match (case-sensitive)
  5. Error code string values: exact match
  6. State enum values: exact match

- **FR-018**: Mismatches SHALL be flagged as findings:
  ```
  ### Finding: SPEC-CONTRACT-XXX
  - **Severity**: HIGH
  - **Category**: interface-mismatch | schema-mismatch | api-mismatch | state-mismatch | error-mismatch
  - **Contract file**: <path>
  - **Implementation file**: <path>:<line>
  - **Expected**: <contract definition>
  - **Actual**: <implementation definition>
  - **Recommendation**: Align implementation with contract
  ```

- **FR-019**: If contract files do not exist for a WP (e.g., Planner V1 was used), the skill SHALL fall back to prose-only review and note the absence of contracts as an informational finding.

#### Implementation Contract -- review-spec (expanded)

**Inputs**:
- Implementation source files (from WP scope).
- Spec file path.
- Contract files directory for the WP.

**Outputs**:
- Findings list (prose-based + contract-based).
- Verdict: PASS or FAIL.

**Error behaviors**:
- Contract files missing: fall back to prose-only review, INFO finding.
- Implementation files missing: flag as HIGH finding (unimplemented requirements).

---

## 5. User Stories

### US-01 -- Pre-Planning Spec Validation (Priority: P1) MVP

**As the** Planner agent, **I want** the spec validated for implementation completeness before I decompose it, **so that** I do not plan against a shallow spec that causes downstream rework.

**Why P1**: This is the primary gate that prevents the root cause of pipeline drift.

**Independent Test**: Provide a spec missing error behaviors for 5 FRs. Run review-spec-completeness. Verify: 5 HIGH findings for missing error behaviors, verdict FAIL.

**Acceptance Scenarios**:
1. **Given** a spec where FR-003 uses "should" instead of "SHALL", **When** review-spec-completeness runs, **Then** SPEC-COMP-001 is reported with severity HIGH, category obligation-language.
2. **Given** a spec where entity User is missing the "role" field type, **When** the skill runs, **Then** a HIGH finding for untyped field is reported.
3. **Given** a spec where Section 16 traceability matrix has FR-012 with no US mapping, **When** the skill runs, **Then** a HIGH finding for empty traceability cell is reported.
4. **Given** a spec that passes all checks, **When** the skill runs, **Then** verdict is PASS with zero HIGH findings.

---

### US-02 -- Contract-Aware Code Review (Priority: P1) MVP

**As the** Review Coordinator, **I want** review-spec to validate code against contract files, **so that** implementation drift from formal contracts is caught immediately.

**Why P1**: Contract-aware review is the enforcement mechanism for contract-first implementation.

**Independent Test**: Implement a function with a different parameter name than the contract. Run review-spec. Verify: a HIGH finding for interface-mismatch is reported.

**Acceptance Scenarios**:
1. **Given** contract defines `createUser(input: CreateUserInput)` and implementation has `createUser(data: CreateUserInput)`, **When** review-spec runs, **Then** a SPEC-CONTRACT finding reports parameter name mismatch ("input" vs "data").
2. **Given** contract defines `User.email: string` and implementation has `User.emailAddress: string`, **When** review-spec runs, **Then** a SPEC-CONTRACT finding reports field name mismatch.
3. **Given** no contract files exist for the WP, **When** review-spec runs, **Then** it falls back to prose-only review with an INFO finding about missing contracts.

---

### Edge Cases

- What happens when a contract file has syntax errors? The skill flags the syntax error as a HIGH finding and cannot perform contract-based checks for that file.
- What happens when the implementation has extra functions not in the contract? This is acceptable if the extra functions are private/internal. Public functions not in the contract are flagged as MEDIUM findings.
- What happens when the spec has no companion artifacts? review-spec-completeness flags this as HIGH (artifacts expected for V2 specs). review-spec falls back to prose-only.

---

## 6. User Flows

### 6.1 Pre-Planning Gate Flow

1. Planner coordinator runs completeness pre-check (or Review Coordinator dispatches the skill).
2. review-spec-completeness reads spec + artifacts.
3. Skill runs all checks (FR-003 through FR-012).
4. Skill produces findings and verdict.
5. If FAIL: findings passed to Planner, which auto-loops to Spec Architect.
6. If PASS: Planner proceeds to decomposition.

### 6.2 Contract-Aware Review Flow

1. Review Coordinator dispatches review-spec for a WP.
2. review-spec reads implementation files, spec, and contract files.
3. Skill performs prose-based checks (existing behavior).
4. Skill performs contract-based checks (FR-016 through FR-018).
5. Skill produces combined findings and verdict.
6. Review Coordinator aggregates with other skill findings.

---

## 7. Data Model

### 7.1 Completeness Finding

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | string | SPEC-COMP-XXX | Unique finding identifier |
| severity | enum | HIGH, MEDIUM, LOW | Impact level |
| category | enum | obligation-language, error-behavior, data-model, api-contract, state-machine, traceability, integration, ambiguity, artifact-consistency, security | Finding type |
| location | string | "Section N, FR-XXX" or entity name | Where in the spec |
| issue | string | 1-500 chars | What is wrong |
| recommendation | string | 1-500 chars | How to fix |

### 7.2 Contract Finding

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | string | SPEC-CONTRACT-XXX | Unique finding identifier |
| severity | enum | HIGH, MEDIUM, LOW | Impact level |
| category | enum | interface-mismatch, schema-mismatch, api-mismatch, state-mismatch, error-mismatch | Contract check type |
| contract_file | string | valid path | Contract file path |
| impl_file | string | valid path:line | Implementation file path and line |
| expected | string | from contract | What the contract defines |
| actual | string | from implementation | What the code implements |
| recommendation | string | 1-500 chars | How to fix |

### 7.3 Verdict

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| result | enum | PASS, FAIL | Overall assessment |
| high_count | integer | >= 0 | Number of HIGH findings |
| medium_count | integer | >= 0 | Number of MEDIUM findings |
| low_count | integer | >= 0 | Number of LOW findings |

---

## 8. API / Interface Design

### 8.1 Skill File Interfaces

Both skills follow the standard Review Coordinator skill contract:
- Discovered via glob pattern `review-*/SKILL.md`
- Dispatched as subagent by the Review Coordinator
- Produce findings in the standard format
- Return a PASS/FAIL verdict

### 8.2 review-spec-completeness Dispatch Prompt

```
Validate spec completeness for planning readiness.

1. Read skill instructions at: .github/skills/review-spec-completeness/SKILL.md
2. Read the spec at: <spec_path>
3. Read companion artifacts at: <artifacts_dir>

Check every FR for SHALL language, error behavior, data model completeness,
API contract completeness, state machines, traceability, integration strategies,
ambiguity, and artifact consistency.

Produce findings in standard format. Verdict: PASS (0 HIGH) or FAIL (1+ HIGH).
```

### 8.3 review-spec Expanded Dispatch Prompt

```
Review implementation against spec and contracts.

1. Read skill instructions at: .github/skills/review-spec/SKILL.md
2. Read implementation files: <file_list>
3. Read spec at: <spec_path>
4. Read contract files at: <contracts_dir>

Perform prose-based review (existing behavior).
Additionally, perform contract-aware checks:
- Compare function signatures against interfaces.<ext>
- Compare entity fields against data-schemas.<ext>
- Compare API endpoints against api-contracts.<ext>
- Compare state transitions against state-machines.<ext>
- Compare error codes against error-catalog.<ext>

Produce findings. Verdict: PASS or FAIL.
```

---

## 9. Architecture

### 9.1 System Design

Both skills integrate into the existing Review Coordinator architecture:

```
Review Coordinator
       |
       |--> Discover review skills (review-*/)
       |
       |--> review-spec-completeness  (NEW: pre-planning gate)
       |--> review-spec               (EXPANDED: + contract checks)
       |--> review-security           (existing)
       |--> review-quality            (existing)
       |--> review-tests              (existing)
       |--> review-architecture       (existing)
       |--> review-performance        (existing)
       |--> review-docs               (existing)
       |--> review-deps               (existing)
```

### 9.2 Technology Stack

| Layer | Technology | Version | Rationale |
|-------|-----------|---------|-----------|
| Skill framework | VS Code Copilot Chat skills | Current | Existing review skill architecture |
| Discovery | glob pattern `review-*/SKILL.md` | N/A | Existing dynamic discovery |

### 9.3 Directory & Module Structure

```
.github/
  skills/
    review-spec-completeness/SKILL.md  # NEW: pre-planning validation gate
    review-spec/SKILL.md               # MODIFIED: add contract-aware checks
```

### 9.4 Key Design Decisions

**Decision 1: review-spec-completeness as a review skill (not standalone agent)**
- **Rationale**: Fits the existing review skill architecture. No new agent type needed. Can be dispatched by Review Coordinator or directly by Planner.
- **Alternatives considered**: Standalone validation agent, built into Planner, Spec Architect self-validates.
- **Consequences**: Reusable across contexts. The Planner can invoke it directly for its pre-check.

**Decision 2: Expand review-spec (not create review-contracts)**
- **Rationale**: Contract checking IS spec adherence checking. Creating a separate skill would be artificial separation. The review-spec skill's purpose is "does code match spec" -- contracts are simply a more precise representation of the spec.
- **Alternatives considered**: New review-contracts skill, each review skill checks its own contracts.
- **Consequences**: review-spec becomes the comprehensive spec+contract checker. No skill proliferation.

---

## 10. Non-Functional Requirements

### 10.1 Performance
- **NFR-001**: review-spec-completeness SHALL complete within 5 minutes for a spec up to 3000 lines.
- **NFR-002**: review-spec (with contract checks) SHALL complete within 8 minutes for a medium WP (800 lines of implementation, 6 contract files).

### 10.2 Security
- No special security requirements. Skills read files, produce findings.

### 10.3 Scalability & Availability
- Local workspace only.

---

## 11. Test Requirements

### 11.2 BDD / Acceptance Tests

```gherkin
Feature: review-spec-completeness - Pre-Planning Validation

  Scenario: Flag FRs with weak obligation language
    Given a spec where FR-003 uses "should"
    When review-spec-completeness runs
    Then finding SPEC-COMP-001 is reported with severity HIGH
    And category is "obligation-language"

  Scenario: Flag missing error behavior
    Given a spec where FR-005 has no error behavior defined
    When the skill runs
    Then a HIGH finding for missing error behavior is reported
    And the recommendation suggests adding error behavior

  Scenario: Flag incomplete data model
    Given a spec where entity User has field "role" with no type
    When the skill runs
    Then a HIGH finding for untyped field is reported

  Scenario: Flag empty traceability cell
    Given Section 16 has FR-012 with no US mapping
    When the skill runs
    Then a HIGH finding for empty traceability cell is reported

  Scenario: Pass a complete spec
    Given a spec where all checks pass
    When the skill runs
    Then verdict is PASS with 0 HIGH findings

Feature: review-spec - Contract-Aware Review

  Scenario: Detect function signature mismatch
    Given contract defines createUser(input: CreateUserInput)
    And implementation has createUser(data: CreateUserInput)
    When review-spec runs with contract files
    Then SPEC-CONTRACT finding for interface-mismatch is reported
    And expected is "input" and actual is "data"

  Scenario: Detect field name mismatch
    Given contract defines User.email and implementation has User.emailAddress
    When review-spec runs
    Then SPEC-CONTRACT finding for schema-mismatch is reported

  Scenario: Fallback to prose-only when no contracts
    Given no contract files exist for the WP
    When review-spec runs
    Then it performs prose-only review
    And reports an INFO finding about missing contracts

  Scenario: Detect missing error code
    Given contract error catalog has USR-001 and USR-002
    And implementation only handles USR-001
    When review-spec runs
    Then SPEC-CONTRACT finding for error-mismatch is reported for USR-002
```

---

## 12. Constraints & Assumptions

### Constraints
- Must integrate with the existing Review Coordinator's dynamic discovery (no coordinator changes).
- Skills follow the standard review skill contract (findings format, PASS/FAIL verdict).

### Assumptions
1. Reviewer V2 (Spec 001) is implemented with dynamic skill discovery.
2. Contract files are available in `.sdd/plans/contracts/` when the Coder has been used.
3. Older specs without companion artifacts are handled gracefully with fallback behavior.

---

## 13. Out of Scope

- **Fixing spec issues**: The skill flags issues; the Spec Architect fixes them.
- **Fixing implementation issues**: The skill flags issues; the Coder fixes them.
- **Non-spec review dimensions**: Security, quality, performance, etc. are handled by their dedicated review skills.

---

## 14. Open Questions

None remaining.

---

## 15. Glossary

- **Completeness check**: Validation that a spec has sufficient depth for autonomous implementation.
- **Contract-aware review**: Comparing implementation code against language-specific contract files for exact match.
- **Pre-planning gate**: A validation step that must pass before the Planner decomposes the spec.

---

## 16. Traceability Matrix

| FR ID | Requirement Summary | User Story | Acceptance Scenario | Test Type | Test Section Ref |
|-------|-------------------|------------|--------------------|-----------|----|
| FR-001 | Completeness skill purpose | US-01 | Scenario 4 | BDD | 11.2 |
| FR-003 | SHALL language check | US-01 | Scenario 1 | BDD | 11.2 |
| FR-004 | Error behavior check | US-01 | Scenario 2 | BDD | 11.2 |
| FR-005 | Data model completeness | US-01 | Scenario 3 | BDD | 11.2 |
| FR-006 | API endpoint completeness | US-01 | Scenario 2 | BDD | 11.2 |
| FR-007 | State machine completeness | US-01 | Scenario 4 | BDD | 11.2 |
| FR-008 | Traceability matrix check | US-01 | Scenario 3 | BDD | 11.2 |
| FR-010 | Ambiguity check | US-01 | Scenario 4 | BDD | 11.2 |
| FR-011 | Artifact consistency check | US-01 | Scenario 4 | BDD | 11.2 |
| FR-016 | Contract-aware code checks | US-02 | Scenario 1, 2 | BDD | 11.2 |
| FR-017 | Token-by-token comparison | US-02 | Scenario 1 | BDD | 11.2 |
| FR-018 | Contract mismatch findings | US-02 | Scenario 1, 2 | BDD | 11.2 |
| FR-019 | Fallback to prose-only | US-02 | Scenario 3 | BDD | 11.2 |

---

## 17. Technical References

### Review Architecture
- Reviewer V2 Spec (001-reviewer-v2), `.sdd/specs/001-reviewer-v2-skill-based-architecture.spec.md`, consulted 2026-04-05

---

## 18. Version History

| Version | Date | Author | Summary of Changes |
|---------|------|--------|--------------------|
| 1.0 | 2026-04-05 | Spec Architect | Initial specification |
| 1.0.1 | 2026-04-05 | Spec Architect | Self-review corrections: verified findings format consistency, confirmed traceability matrix completeness |
