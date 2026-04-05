# Agent Handoff Schemas & Domain-Specific Patterns -- Specification

> **Source brief**: `.sdd/ideas/002-sdd-pipeline-v2-universal-skill-architecture.md`
> **Feature branch**: `006-handoff-schemas-patterns`
> **Status**: Validated
> **Version**: 1.1

---

## 1. Overview

Formalize agent-to-agent handoff contracts as schema files in `.github/schemas/` and restructure the review patterns system from a single `review-patterns.md` into per-domain pattern files (`spec-patterns.md`, `plan-patterns.md`, `code-patterns.md`, `doc-patterns.md`). Handoff schemas define exactly what each agent produces and what the downstream agent expects, eliminating interpretation-based drift at handoff points. Domain-specific patterns capture recurring issues per domain (spec writing, planning, coding, documentation) and are consumed by the relevant agent at startup before any skill runs.

---

## 2. Goals & Success Criteria

- **SC-001**: Every agent handoff has a corresponding schema defining required inputs, outputs, and validation rules. Verified by: one schema file per handoff point.
- **SC-002**: Schema validation prevents an agent from starting with incomplete or malformed input from the prior agent. Verified by: coordinators validate handoff inputs against the schema before proceeding.
- **SC-003**: Each domain has its own patterns file consumed only by the relevant agent. Verified by: Spec Architect reads `spec-patterns.md`, Planner reads `plan-patterns.md`, Coder reads `code-patterns.md`, Docs Agent reads `doc-patterns.md`.
- **SC-004**: Patterns are updated by the Reviewer when recurring findings emerge. Verified by: Review Coordinator appends new patterns to the domain-specific file after reviews.

---

## 3. Users & Roles

- **All Pipeline Agents (consumers)**: Read their domain's patterns at startup. Validate handoff inputs against schemas before proceeding.
- **Review Coordinator (patterns curator)**: Appends new patterns to domain-specific files when recurring findings emerge across reviews.
- **System Maintainer (schema maintainer)**: Creates and updates handoff schemas when agent interfaces change.

---

## 4. Functional Requirements

### 4.1 Agent Handoff Schemas

#### 4.1.1 Schema Files

- **FR-001**: The system SHALL define handoff schemas as YAML files in `.github/schemas/`, one per directional handoff:
  1. `ideation-to-spec.schema.yaml` - Ideation/Brainstorming -> Spec Architect
  2. `spec-to-planner.schema.yaml` - Spec Architect -> Planner
  3. `planner-to-coder.schema.yaml` - Planner -> Coder
  4. `coder-to-reviewer.schema.yaml` - Coder -> Review Coordinator
  5. `reviewer-to-coder.schema.yaml` - Review Coordinator -> Coder (rework)
  6. `reviewer-to-spec.schema.yaml` - Review Coordinator -> Spec Architect (spec gaps)
  7. `planner-to-spec.schema.yaml` - Planner -> Spec Architect (auto-loop)
  8. `orchestrator-handoff.schema.yaml` - Orchestrator -> any agent
  - Error: If any listed schema file is missing from `.github/schemas/`, the system SHALL report the missing file(s) at startup.

- **FR-002**: Each schema file SHALL define:
  1. `source_agent`: The producing agent name (string)
  2. `target_agent`: The consuming agent name (string)
  3. `required_artifacts`: List of file paths or glob patterns the source MUST produce before handoff
  4. `required_state`: Conditions that must be true (e.g., spec status = Validated, WP lane = for_review)
  5. `context_fields`: Key-value pairs that must be present in the handoff prompt (e.g., spec_path, wp_id)
  6. `validation_rules`: Checks the target agent runs before accepting the handoff
  - Error: If a schema file omits any required field, the coordinator SHALL halt with a validation error listing the missing fields.

#### 4.1.2 Schema Format

- **FR-003**: Each schema SHALL follow this YAML structure:
  ```yaml
  schema: handoff/v1
  source_agent: "2. Spec Architect"
  target_agent: "3. Planner"
  description: "Handoff from Spec Architect to Planner after spec validation"

  required_artifacts:
    - path: ".sdd/specs/{NNN}-{name}.spec.md"
      validation:
        - field: "Status"
          value: "Validated"
        - exists: true

    - path: ".sdd/specs/artifacts/{NNN}-{name}/"
      validation:
        - min_files: 1

  required_state:
    - condition: "spec.status == 'Validated'"
      error: "Spec must be Validated before planning"

  context_fields:
    - name: "spec_path"
      type: "string"
      required: true
      description: "Path to the validated spec file"

    - name: "artifacts_dir"
      type: "string"
      required: true
      description: "Path to companion artifacts directory"

  validation_rules:
    - check: "file_exists"
      target: "spec_path"
      error: "Spec file does not exist at {spec_path}"

    - check: "field_value"
      target: "spec_path"
      field: "Status"
      expected: "Validated"
      error: "Spec status is not Validated"
  ```
  - Error: If a schema file fails YAML parsing or lacks required top-level keys (`schema`, `source_agent`, `target_agent`, `required_artifacts`, `context_fields`), the coordinator SHALL halt and report the file path and parse error.

#### 4.1.3 Schema Validation at Handoff

- **FR-004**: Each agent coordinator SHALL validate incoming handoff against the relevant schema before proceeding. Validation SHALL:
  1. Check `required_artifacts` exist and pass their validation rules
  2. Check `required_state` conditions are met
  3. Check `context_fields` are present and have valid values
  4. Run `validation_rules` checks
  - Error: If any validation fails, the coordinator SHALL halt and report which checks failed with the schema's error messages.

- **FR-005**: Schema validation SHALL be the FIRST action a coordinator performs after receiving a handoff, before any research or skill dispatch.
  - Error: If a coordinator performs research or skill dispatch before schema validation completes, the handoff is non-compliant and all output SHALL be discarded.

#### 4.1.4 Schema Maintenance

- **FR-006**: When an agent's interface changes (e.g., new required artifacts, new context fields), the corresponding schema file SHALL be updated. Schema changes SHALL be committed with the agent changes.
  - Error: If schema and agent interface diverge, downstream handoff validation SHALL fail with stale contract errors.

- **FR-007**: Schemas SHALL be versioned via the `schema: handoff/v1` header. Future incompatible changes SHALL increment the version (v2, v3).
  - Error: If a coordinator encounters an unrecognized schema version, it SHALL halt with "Unsupported schema version: {version}".

#### Implementation Contract -- Handoff Schemas

**Inputs**: Schema files in `.github/schemas/`.
**Outputs**: Validation pass/fail at each handoff point.
**Error behaviors**: Schema validation failure - halt, report failed checks with error messages.

---

### 4.2 Domain-Specific Patterns

#### 4.2.1 Pattern Files

- **FR-008**: The system SHALL maintain domain-specific pattern files in `.sdd/reviews/`:
  1. `spec-patterns.md` - Patterns for specification writing (consumed by Spec Architect)
  2. `plan-patterns.md` - Patterns for planning and decomposition (consumed by Planner)
  3. `code-patterns.md` - Patterns for implementation (consumed by Coder)
  4. `doc-patterns.md` - Patterns for documentation (consumed by Docs Agent)
  - Each file replaces the domain-relevant entries from the former single `review-patterns.md`.
  - Error: If a domain pattern file does not exist, the consuming agent SHALL proceed without patterns and log a warning.

#### 4.2.2 Pattern Format

- **FR-009**: Each pattern file SHALL follow this structure:
  ```markdown
  # [Domain] Patterns

  ## Active Patterns

  ### PAT-[DOMAIN]-XXX: [Pattern Title]
  - **Status**: active | retired | superseded
  - **Added**: YYYY-MM-DD
  - **Source**: Review of WP<NN> / Spec <NNN> / manual
  - **Trigger**: [What symptoms indicate this pattern is occurring]
  - **Prevention**: [What the agent should do to avoid this pattern]
  - **Example**: [Concrete example of the pattern and its fix]

  ## Retired Patterns

  [Patterns that no longer apply, kept for historical reference]
  ```
  - Error: If a pattern entry lacks any required field (id, title, status, added, source, trigger, prevention), the consuming agent SHALL skip that entry and log a warning.

- **FR-010**: Pattern IDs SHALL use domain prefixes:
  - `PAT-SPEC-XXX` for spec patterns
  - `PAT-PLAN-XXX` for plan patterns
  - `PAT-CODE-XXX` for code patterns
  - `PAT-DOC-XXX` for doc patterns
  - Error: If a pattern ID does not match the required `PAT-{DOMAIN}-XXX` format, the consuming agent SHALL skip the entry and log a warning.

#### 4.2.3 Pattern Consumption

- **FR-011**: Each agent coordinator SHALL read its domain-specific patterns file at startup (before any skill dispatch). Active patterns SHALL be included in the prompt for every skill that agent dispatches.
  - Error: If the patterns file cannot be parsed, the agent SHALL proceed without patterns and log the parse error.

- **FR-012**: Patterns from other domains SHALL NOT be included. The Spec Architect reads only `spec-patterns.md`, the Planner reads only `plan-patterns.md`, the Coder reads only `code-patterns.md`, and the Docs Agent reads only `doc-patterns.md`.
  - Error: If cross-domain patterns are detected in an agent's prompt, the coordinator SHALL strip them before skill dispatch.

#### 4.2.4 Pattern Curation

- **FR-013**: The Review Coordinator SHALL track finding recurrence across reviews. When the same finding category appears in 3 or more reviews:
  1. Create a new pattern entry in the relevant domain-specific file
  2. Set status to "active"
  3. Include the trigger, prevention, and example from the recurring findings
  - Error: If the Review Coordinator cannot determine the target domain for a recurring finding, it SHALL place the pattern in the closest-matching domain file with a `[NEEDS REVIEW]` tag.

- **FR-014**: The Review Coordinator SHALL retire patterns that have not been triggered in 10 consecutive reviews:
  1. Move the pattern to the "Retired Patterns" section
  2. Set status to "retired"
  3. Add a retirement date
  - Error: If review count tracking is unavailable, retirement processing SHALL be deferred until tracking data is restored.

- **FR-015**: Pattern curation SHALL be committed:
  ```
  git add .sdd/reviews/<domain>-patterns.md
  git commit -m "docs(patterns): add PAT-<DOMAIN>-XXX <pattern title>"
  ```
  - Error: If the commit fails, the coordinator SHALL retry once and report the failure if it persists.

#### Implementation Contract -- Domain-Specific Patterns

**Inputs**: Finding history from reviews.
**Outputs**: Pattern files in `.sdd/reviews/`.
**Error behaviors**: Pattern file does not exist - agent proceeds without patterns, logs a warning.

---

### 4.3 Migration from Single Patterns File

- **FR-016**: If `.sdd/reviews/review-patterns.md` exists (legacy single file), the system SHALL:
  1. Read all patterns from the legacy file
  2. Categorize each pattern into its domain (spec, plan, code, doc) based on the pattern's content and trigger
  3. Write each pattern to its categorized domain-specific file
  4. Rename the legacy file to `review-patterns.md.bak`
  - The migration SHALL be idempotent: running it when domain files already exist does not duplicate patterns.
  - Error: If the legacy file cannot be parsed, the migration SHALL halt and report the parse error. If a pattern with the same ID already exists in the target domain file, the duplicate SHALL be skipped.

---

## 5. User Stories

### US-01 -- Schema-Validated Handoffs (Priority: P1) MVP

**As a** pipeline agent, **I want** my inputs validated against a schema before I start work, **so that** I never process incomplete or malformed input from the prior agent.

**Why P1**: Schema validation prevents the most common handoff failures.

**Independent Test**: Attempt to start the Planner with a spec that has status "Draft". Verify: schema validation fails, Planner halts with clear error message.

**Acceptance Scenarios**:
1. **Given** a validated spec with companion artifacts, **When** the Planner receives the handoff, **Then** schema validation passes and planning proceeds.
2. **Given** a spec with status "Draft", **When** the Planner receives the handoff, **Then** schema validation fails with error "Spec must be Validated before planning".
3. **Given** a WP with no implementation files, **When** the Reviewer receives the handoff, **Then** schema validation fails with error about missing implementation.

---

### US-02 -- Domain-Specific Pattern Consumption (Priority: P1) MVP

**As a** pipeline agent, **I want** to read patterns specific to my domain before generating output, **so that** I avoid repeating known mistakes.

**Why P1**: Patterns encode institutional knowledge that prevents recurring issues.

**Independent Test**: Add a pattern to `spec-patterns.md` about always defining error behavior. Run the Spec Architect. Verify: the generated spec includes error behavior for every FR.

**Acceptance Scenarios**:
1. **Given** `spec-patterns.md` has an active pattern "always define error behavior for FRs", **When** the Spec Architect runs, **Then** every FR in the generated spec has error behavior.
2. **Given** `code-patterns.md` has an active pattern "always validate input length", **When** the Coder implements, **Then** input validation includes length checks.
3. **Given** no patterns file exists for a domain, **When** the agent runs, **Then** it proceeds without patterns and logs a warning.

---

### US-03 -- Automated Pattern Curation (Priority: P2)

**As the** Review Coordinator, **I want** to automatically create patterns from recurring findings, **so that** the same issues are prevented in future iterations.

**Why P2**: Reduces manual pattern management.

**Independent Test**: Submit 3 reviews with the same finding category. Verify: a new pattern is created in the relevant domain file.

**Acceptance Scenarios**:
1. **Given** 3 reviews flagged "missing error behavior in FRs", **When** the Review Coordinator curates patterns, **Then** PAT-SPEC-001 is created in `spec-patterns.md`.
2. **Given** PAT-CODE-005 has not been triggered in 10 reviews, **When** the Review Coordinator curates, **Then** the pattern is moved to "Retired".

---

### Edge Cases

- What happens when a schema references artifacts that should exist but the agent produced zero? Validation fails, handoff blocked.
- What happens when the legacy patterns file has uncategorizable patterns? Place them in `code-patterns.md` as the default domain with a note for manual review.

---

## 6. User Flows

### 6.1 Schema Validation at Handoff

1. Source agent completes its work and produces artifacts.
2. Orchestrator (or source agent) initiates handoff to target agent.
3. Target agent coordinator reads the handoff schema from `.github/schemas/`.
4. Coordinator checks required_artifacts, required_state, context_fields, validation_rules.
5. All pass: proceed to agent's work.
6. Any fail: halt, report validation errors, recommend fixing the source agent's output.

### 6.2 Pattern Curation Flow

1. Review Coordinator completes a review with findings.
2. Coordinator checks finding categories against historical finding data.
3. If a category appears in 3+ reviews: create a new pattern in the domain file.
4. If a pattern has not been triggered in 10 reviews: retire it.
5. Commit pattern file changes.

---

## 7. Data Model

### 7.1 Handoff Schema

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| schema | string | "handoff/v1" | Schema format version |
| source_agent | string | valid agent name | Producing agent |
| target_agent | string | valid agent name | Consuming agent |
| description | string | 1-200 chars | Purpose of this handoff |
| required_artifacts | list(ArtifactSpec) | min 1 | Files that must exist |
| required_state | list(StateCondition) | optional | State conditions |
| context_fields | list(ContextField) | min 1 | Prompt fields |
| validation_rules | list(ValidationRule) | optional | Additional checks |

### 7.2 Pattern Entry

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | string | PAT-{DOMAIN}-XXX | Unique pattern ID |
| title | string | 1-100 chars | Brief description |
| status | enum | active, retired, superseded | Current state |
| added | date | YYYY-MM-DD | When created |
| source | string | review ref or "manual" | Origin |
| trigger | string | 1-500 chars | Symptom description |
| prevention | string | 1-500 chars | How to avoid |
| example | string | optional | Concrete instance |

---

## 8. API / Interface Design

Schemas and patterns are file-based. No runtime API.

---

## 9. Architecture

### 9.1 Directory Structure

```
.github/
  schemas/
    ideation-to-spec.schema.yaml
    spec-to-planner.schema.yaml
    planner-to-coder.schema.yaml
    coder-to-reviewer.schema.yaml
    reviewer-to-coder.schema.yaml
    reviewer-to-spec.schema.yaml
    planner-to-spec.schema.yaml
    orchestrator-handoff.schema.yaml

.sdd/
  reviews/
    spec-patterns.md     # NEW (replaces spec entries from review-patterns.md)
    plan-patterns.md     # NEW
    code-patterns.md     # NEW
    doc-patterns.md      # NEW
```

### 9.2 Key Design Decisions

**Decision 1: YAML for schemas (not JSON or Markdown)**
- **Rationale**: YAML is human-readable, supports comments, and is concise for configuration. JSON lacks comments. Markdown is too loose for structured validation.
- **Alternatives**: JSON Schema, Markdown tables.
- **Consequences**: Agents must parse YAML, which is standard for config in this ecosystem.

**Decision 2: One schema per directional handoff (not per agent)**
- **Rationale**: Handoffs are directional -- what Planner sends to Coder differs from what it sends to Spec Architect. Per-direction schemas capture exact contracts.
- **Alternatives**: One schema per agent (all inputs), one mega-schema.
- **Consequences**: 8 schema files. Manageable.

**Decision 3: Pattern files per domain (not per agent)**
- **Rationale**: Domains map cleanly to agent responsibilities. "Spec patterns" are consumed by Spec Architect; "code patterns" by Coder.
- **Alternatives**: One file per agent, single file with tags, embedded in agent files.
- **Consequences**: Clear separation. Each agent reads one file.

---

## 10. Non-Functional Requirements

### 10.1 Performance

- **NFR-001**: Schema validation SHALL complete in under 5 seconds (file existence checks + field parsing).
- **NFR-002**: Pattern files SHALL remain under 200 entries per domain to avoid context bloat.

### 10.2 Security

- **NFR-003**: Schema files and pattern files SHALL NOT contain executable code or template expressions. Agents SHALL treat these files as declarative configuration only.
  - Error: If a schema or pattern file contains executable expressions, the coordinator SHALL halt and report "Untrusted content detected in {file_path}".

---

## 11. Test Requirements

### 11.1 Test Coverage

All schema validation logic and pattern file parsing SHALL have unit tests covering valid inputs, invalid inputs, and edge cases. BDD scenarios below cover acceptance-level behavior.

### 11.2 BDD / Acceptance Tests

```gherkin
Feature: Handoff Schema Validation

  Scenario: Valid handoff passes schema
    Given a validated spec with companion artifacts
    When spec-to-planner schema is validated
    Then validation passes

  Scenario: Invalid handoff fails schema
    Given a spec with status "Draft"
    When spec-to-planner schema is validated
    Then validation fails with "Spec must be Validated before planning"

Feature: Domain-Specific Patterns

  Scenario: Agent reads domain patterns at startup
    Given spec-patterns.md has 3 active patterns
    When the Spec Architect starts
    Then all 3 patterns are included in skill prompts

  Scenario: Agent ignores other domain patterns
    Given code-patterns.md has 5 active patterns
    When the Spec Architect starts
    Then code patterns are NOT included in prompts

  Scenario: Pattern curation on recurring finding
    Given 3 reviews flagged "missing error behavior"
    When the Review Coordinator curates
    Then PAT-SPEC-001 is created in spec-patterns.md
```

---

## 12. Constraints & Assumptions

### Constraints
- Schemas are static files, not runtime APIs.
- Pattern files must fit within agent context windows (200 entries max).

### Assumptions
1. All agents are refactored to coordinator pattern (Specs 002-004).
2. Review Coordinator tracks finding history for pattern curation.
3. YAML parsing is supported by the LLM reading the schema files.

---

## 13. Out of Scope

- **Schema auto-generation**: Schemas are manually authored and maintained.
- **Schema evolution tooling**: No automated migration tooling for schema version changes.
- **Pattern ML/analytics**: Patterns are manually or rule-based curated, not ML-driven.

---

## 14. Open Questions

None remaining.

---

## 15. Glossary

- **Handoff schema**: A YAML file defining the contract between two agents at a handoff point.
- **Domain pattern**: A documented recurring issue with trigger, prevention, and example, scoped to a domain.
- **Pattern curation**: The process of creating and retiring patterns based on review finding frequency.

---

## 16. Traceability Matrix

| FR ID | Requirement Summary | User Story | Acceptance Scenario | Test Type | Test Section Ref |
|-------|-------------------|------------|--------------------|-----------|----|
| FR-001 | Schema files per handoff | US-01 | Scenario 1, 2 | BDD | 11.2 |
| FR-002 | Schema structure definition | US-01 | Scenario 1 | BDD | 11.2 |
| FR-003 | Schema YAML format | US-01 | Scenario 1 | BDD | 11.2 |
| FR-004 | Schema validation at handoff | US-01 | Scenario 1, 2, 3 | BDD | 11.2 |
| FR-005 | Validation before any other action | US-01 | Scenario 2 | BDD | 11.2 |
| FR-006 | Schema updates with interface changes | US-01 | Scenario 1 | BDD | 11.2 |
| FR-007 | Schema versioning | US-01 | Scenario 1 | BDD | 11.2 |
| FR-008 | Domain-specific pattern files | US-02 | Scenario 1, 2 | BDD | 11.2 |
| FR-009 | Pattern file format | US-02 | Scenario 1 | BDD | 11.2 |
| FR-010 | Pattern ID domain prefixes | US-02 | Scenario 1 | BDD | 11.2 |
| FR-011 | Pattern consumption at startup | US-02 | Scenario 1, 3 | BDD | 11.2 |
| FR-012 | No cross-domain patterns | US-02 | Scenario 2 | BDD | 11.2 |
| FR-013 | Automated pattern curation | US-03 | Scenario 1 | BDD | 11.2 |
| FR-014 | Pattern retirement | US-03 | Scenario 2 | BDD | 11.2 |
| FR-015 | Pattern curation commit format | US-03 | Scenario 1 | BDD | 11.2 |
| FR-016 | Legacy patterns migration | US-02 | Scenario 1 | BDD | 11.2 |

---

## 17. Technical References

### Standards
- YAML 1.2 Specification, https://yaml.org/spec/1.2.2/, consulted 2026-04-05

---

## 18. Version History

| Version | Date | Author | Summary of Changes |
|---------|------|--------|--------------------|
| 1.0 | 2026-04-05 | Spec Architect | Initial specification |
| 1.1 | 2026-04-05 | Spec Architect | Add error behaviors to all FRs; complete traceability matrix; add security NFRs; add companion artifacts; promote to Validated |
