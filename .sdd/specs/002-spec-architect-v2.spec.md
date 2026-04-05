# Spec Architect V2 - Skill-Based Deep Specification -- Specification

> **Source brief**: `.sdd/ideas/002-sdd-pipeline-v2-universal-skill-architecture.md`
> **Feature branch**: `002-spec-architect-v2`
> **Status**: Approved
> **Version**: 1.0

---

## 1. Overview

Replace the monolithic Spec Architect agent (single-pass, 16-section prose-heavy specification) with a skill-based Spec Architect coordinator that dispatches 8 sequential skills, each responsible for one specification domain. Each skill reads and appends to a shared accumulator file (the spec itself), enabling context forwarding between dependent sections. The resulting specification is deeper -- producing 80% of the technical design including language-specific code artifacts (interfaces, schemas, state machines, error catalogs) as companion files alongside the prose spec. This eliminates the primary root cause of pipeline drift: specifications too shallow for autonomous agents to implement faithfully.

---

## 2. Goals & Success Criteria

- **SC-001**: Each spec skill runs in a fresh context window, enabling deeper analysis per section than a single-pass approach. Verified by: each subagent loads only its SKILL.md plus the accumulator file -- no other skill instructions compete for context.
- **SC-002**: Specifications include language-specific companion artifacts (interfaces, data models, API contracts, state machines, error catalogs) that the Planner and Coder can consume without interpretation. Verified by: `.sdd/specs/artifacts/` contains at least one code artifact file per spec.
- **SC-003**: Spec sections produced by later skills (e.g., API Design) are consistent with sections produced by earlier skills (e.g., Data Model) because they read the accumulator. Verified by: no field name, type, or constraint contradictions between sections.
- **SC-004**: The Spec Architect validates its own output before presenting to the user, catching ambiguity and incompleteness. Verified by: the coordinator runs an inline completeness check after all skills finish.
- **SC-005**: Adding a new spec section requires creating exactly one skill file in `.github/skills/spec-*/SKILL.md`. Verified by: no coordinator edit needed to add a spec dimension.
- **SC-006**: Specifications adhere strictly to plain ASCII (no em dashes, smart quotes) and use concrete, testable language (SHALL/SHALL NOT, no "should", "appropriate", "reasonable"). Verified by: encoding and ambiguity checks run post-completion.

---

## 3. Users & Roles

- **Human Developer (primary consumer)**: Reviews and approves the specification at the human gate. Needs the spec to be comprehensive, precise, and free of ambiguity so they can evaluate whether it matches their intent.

- **Planner Agent (downstream consumer)**: Reads the spec to decompose into work packages. Needs: precise FRs with SHALL statements, complete data models with field-level types, API contracts with all error codes, state machines, and a complete traceability matrix.

- **Coder Agent (downstream consumer)**: References the spec during implementation. Needs: companion artifact files (interfaces, schemas) that can be copied verbatim into code, plus prose FRs for behavioral context.

- **Review Coordinator (downstream consumer)**: Uses the spec as the authoritative reference for spec-adherence review. Needs: unambiguous FRs, explicit error behaviors, and measurable success criteria.

- **Orchestrator Agent (invoker)**: Triggers the Spec Architect when a brief exists without a spec. Reads the spec's `Status` field to determine pipeline state.

---

## 4. Functional Requirements

### 4.1 Spec Architect Coordinator

#### 4.1.1 Brief Selection

- **FR-001**: The coordinator SHALL list all files in `.sdd/ideas/` and present them to the user for selection. If only one brief exists, the coordinator SHALL confirm it before proceeding.
  - Error: If `.sdd/ideas/` is empty, the coordinator SHALL inform the user and halt.
  - Error: If the selected brief file is unreadable, the coordinator SHALL halt with a filesystem error.

- **FR-002**: The coordinator SHALL read the selected brief in full before beginning any gap analysis or skill dispatch.

#### 4.1.2 Research Phase

- **FR-003**: The coordinator SHALL invoke a workspace research subagent to discover:
  1. Existing code, configuration, or documentation related to the brief's domain
  2. Existing patterns, frameworks, or conventions already in use
  3. Technical constraints discoverable from the codebase
  - The research subagent SHALL NOT draft any spec content -- discovery and feasibility only.

- **FR-004**: The coordinator SHALL conduct mandatory web research before writing any spec section:
  1. Competing/analogous open-source projects (architecture, API design, data models)
  2. Latest stable versions of all technologies mentioned in the brief
  3. Known pitfalls, anti-patterns, and "lessons learned" for the chosen tech stack
  4. Relevant OWASP entries for the system's threat surface
  5. Relevant standards or RFCs (REST conventions, OAuth2, etc.)

#### 4.1.3 Gap Analysis

- **FR-005**: After research, the coordinator SHALL identify every gap that must be resolved before writing the spec. Gaps SHALL be categorized as:
  - **Functional**: user flows, edge cases, error states, actor permissions
  - **Data & Domain**: entities, attributes, relationships, validation rules
  - **Architecture & Technology**: platform, stack, integrations, deployment
  - **Non-Functional**: performance, security, scalability, accessibility
  - **Testing**: critical behaviors to verify, compliance obligations
  - The coordinator SHALL track gaps using `manage_todo_list`.

- **FR-006**: The coordinator SHALL resolve gaps by asking the user focused questions in batches of no more than 3 questions per turn via `vscode_askQuestions`.
  - If answers significantly change scope, the coordinator SHALL loop back to the research phase.
  - The coordinator SHALL NOT proceed to skill dispatch until all critical gaps are resolved.

#### 4.1.4 Accumulator File Initialization

- **FR-007**: Before dispatching the first skill, the coordinator SHALL create the accumulator file at `.sdd/specs/<NNN>-<idea-name>.spec.md` with:
  1. The spec header (title, source brief, feature branch, status=Draft, version=1.0)
  2. The Overview section (Section 1), written by the coordinator based on the brief and research
  3. The Goals & Success Criteria section (Section 2), derived from the brief's vision and success metrics
  4. The Users & Roles section (Section 3), derived from the brief's target users
  - The coordinator SHALL determine `<NNN>` by checking existing files in `.sdd/specs/` and incrementing.

- **FR-008**: The coordinator SHALL create the companion artifacts directory at `.sdd/specs/artifacts/<NNN>-<idea-name>/` before dispatching any skill that produces code artifacts.

#### 4.1.5 Dynamic Skill Discovery

- **FR-009**: The coordinator SHALL discover available spec skills by scanning for directories matching the glob pattern `.github/skills/spec-*/SKILL.md` at the start of each spec generation.
  - Postcondition: A sorted list of discovered skill names is produced.
  - Error: If zero skills are discovered, the coordinator SHALL halt and report that no spec skills are installed.

- **FR-010**: The coordinator SHALL dispatch discovered skills in a deterministic order. The canonical order is:
  1. `spec-requirements` (functional requirements, NFRs, constraints)
  2. `spec-user-stories` (user stories, acceptance scenarios, edge cases)
  3. `spec-data-model` (entities, fields, types, relationships, validation)
  4. `spec-api-design` (endpoints/interfaces, schemas, error codes)
  5. `spec-architecture` (system design, tech stack, directory structure, decisions)
  6. `spec-security` (security requirements, OWASP mitigations, threat model)
  7. `spec-test-strategy` (test requirements by category, BDD scenarios, coverage)
  8. `spec-traceability` (traceability matrix, cross-reference validation, glossary)
  - Skills not present are skipped. Skills present but not in this list are dispatched after all known skills, in alphabetical order.

#### 4.1.6 Skill Dispatch

- **FR-011**: The coordinator SHALL dispatch each skill as a subagent invocation using `runSubagent`. Each invocation SHALL include:
  1. The skill file path (e.g., `.github/skills/spec-requirements/SKILL.md`)
  2. The accumulator file path (the spec file being built)
  3. The companion artifacts directory path
  4. The source brief file path
  5. Research findings summary (key facts from FR-003 and FR-004)
  6. The spec section number(s) the skill is responsible for
  7. An instruction to read the SKILL.md first, then read the current accumulator state, then write its section(s) to the accumulator file
  - Error: If a subagent invocation fails, the coordinator SHALL halt and report the error. Unlike the Reviewer (which continues on failure), spec skills are sequential and dependent -- a missing section makes later sections invalid.

- **FR-012**: Each skill SHALL execute sequentially, one at a time, blocking. The coordinator SHALL NOT dispatch the next skill until the current skill's subagent returns and has successfully written to the accumulator file.

- **FR-013**: Each skill SHALL read the current state of the accumulator file before writing its section(s). This enables context forwarding: a later skill (e.g., spec-api-design) can reference entities defined by an earlier skill (e.g., spec-data-model).

#### 4.1.7 Companion Artifact Generation

- **FR-014**: Skills that define technical contracts SHALL produce companion artifact files in `.sdd/specs/artifacts/<NNN>-<idea-name>/`. The artifact types are:
  1. **Data model schemas**: Entity definitions in the target language (e.g., TypeScript interfaces, Python dataclasses, SQL DDL)
  2. **API contracts**: Endpoint definitions with typed request/response schemas (e.g., TypeScript types, OpenAPI fragments, Python Protocol classes)
  3. **Interface definitions**: Public function/method signatures with typed parameters and return types
  4. **State machine definitions**: State transition tables or enum-based state definitions in the target language
  5. **Error catalogs**: Error code enums/constants with HTTP status codes and message templates
  6. **Configuration schemas**: Environment variable definitions with types, defaults, and validation rules
  - Each artifact file SHALL include a header comment referencing the source spec section.

- **FR-015**: Companion artifacts SHALL be in the target language specified in the brief/spec's technology stack section. If the target language is not yet determined when a skill runs, the skill SHALL use TypeScript as the default and note the assumption.

- **FR-016**: Companion artifact files SHALL be named descriptively: `data-models.<ext>`, `api-contracts.<ext>`, `interfaces.<ext>`, `state-machines.<ext>`, `error-catalog.<ext>`, `config-schema.<ext>` where `<ext>` matches the target language.

#### 4.1.8 Post-Completion Validation

- **FR-017**: After all skills have completed, the coordinator SHALL run an inline completeness check against the accumulator file:
  1. Every FR uses SHALL or SHALL NOT (no "should", "could", "might")
  2. Every FR has error behavior defined (what happens when the happy path fails)
  3. Every entity in the data model has all fields with types, constraints, and validation rules
  4. Every API endpoint has all response codes (200, 400, 401, 403, 404, 409, 422, 500 as applicable)
  5. Every external integration has a failure strategy (timeout, retry, fallback)
  6. State transitions are explicit for entities with status fields
  7. The traceability matrix has no empty cells
  8. No ambiguous words remain: "appropriate", "reasonable", "as needed", "etc.", "similar"
  9. All `[NEEDS CLARIFICATION]` markers have been resolved
  10. Encoding compliance: no em dashes, smart quotes, curly apostrophes
  - If any check fails, the coordinator SHALL fix the issue inline or ask the user for clarification.

- **FR-018**: After validation, the coordinator SHALL verify that companion artifact files are consistent with the prose spec:
  1. Field names in data model artifacts match Section 7 entity definitions
  2. Endpoint signatures in API artifacts match Section 8 definitions
  3. Error codes in error catalog match Section 4 error behaviors
  4. State values in state machine artifacts match Section 7 state definitions
  - If inconsistencies are found, the coordinator SHALL resolve them before presenting to the user.

#### 4.1.9 Patterns Consumption

- **FR-019**: Before dispatching any skill, the coordinator SHALL read `.sdd/reviews/spec-patterns.md` (if it exists) and include active pattern summaries in the prompt for each skill. Skills SHALL avoid producing spec content that would trigger known patterns.

#### 4.1.10 Presentation and Approval

- **FR-020**: After validation, the coordinator SHALL present the completed spec to the user for review. The file is for persistence; the coordinator SHALL also show the spec content in the chat.

- **FR-021**: On user feedback:
  - Changes requested: revise the spec via targeted skill re-dispatch or inline editing, then re-validate
  - Questions asked: clarify or ask follow-up questions
  - New requirements: loop back to research and gap analysis
  - Approval: change spec status from "Draft" to "Validated", commit, acknowledge

#### 4.1.11 Commit

- **FR-022**: The coordinator SHALL commit the spec and companion artifacts:
  - Include: the spec file (`.sdd/specs/<NNN>-<idea-name>.spec.md`)
  - Include: all files in `.sdd/specs/artifacts/<NNN>-<idea-name>/`
  - Commit message: `docs(spec): add <idea name> specification v1.0`
  - Files SHALL be listed explicitly in `git add`.
  - On revision: `docs(spec): revise <idea name> spec -- <brief description>`

#### Implementation Contract -- Spec Architect Coordinator

**Inputs**:
- Brief file path (string, resolved from `.sdd/ideas/`): the ideation brief to develop into a spec.
- User answers (interactive): responses to gap analysis questions.

**Outputs**:
- Spec file: `.sdd/specs/<NNN>-<idea-name>.spec.md` with all 16+ sections populated.
- Companion artifacts directory: `.sdd/specs/artifacts/<NNN>-<idea-name>/` with language-specific code files.
- Git commit with all files.

**Error behaviors**:
- No briefs in `.sdd/ideas/`: halt, inform user.
- Brief unreadable: halt, report filesystem error.
- Zero spec skills discovered: halt, report no skills installed.
- Skill subagent failure: halt immediately (unlike Reviewer, spec skills are dependent).
- Validation failures: fix inline or ask user; do NOT present incomplete spec.
- Inconsistency between artifacts and prose: resolve before presenting.

---

### 4.2 Spec Skills (Common Contract)

#### 4.2.1 Skill Input Contract

- **FR-023**: Every spec skill SHALL accept the following inputs in its subagent prompt:
  1. `skill_path`: Path to its SKILL.md file
  2. `accumulator_path`: Path to the spec file being built
  3. `artifacts_dir`: Path to the companion artifacts directory
  4. `brief_path`: Path to the source ideation brief
  5. `research_summary`: Key findings from the research phase
  6. `section_numbers`: Which spec sections this skill is responsible for
  7. `patterns`: Active spec-domain patterns to avoid (from spec-patterns.md)
  8. `target_language`: The programming language for companion artifacts

- **FR-024**: Every spec skill SHALL, upon invocation:
  1. Read its own SKILL.md to load its instructions and guidelines
  2. Read the current accumulator file to understand what earlier skills have written
  3. Read the source brief for context
  4. Write its assigned section(s) to the accumulator file (appending, not overwriting prior sections)
  5. Produce companion artifact files in the artifacts directory (if applicable to this skill)

#### 4.2.2 Skill Output Contract

- **FR-025**: Each skill SHALL write its section(s) to the accumulator file using the standard spec template format (numbered headings, FR-XXX identifiers, SHALL statements, implementation contracts).

- **FR-026**: Skills SHALL NOT modify sections written by earlier skills. If a skill discovers an inconsistency with a prior section, it SHALL add an inline marker `[CROSS-REF ISSUE: <description>]` and continue. The coordinator resolves these after all skills complete.

- **FR-027**: Skills SHALL NOT modify the spec header (Section 1, 2, 3) written by the coordinator. Skills own only their assigned section numbers.

- **FR-028**: Skills that produce companion artifacts SHALL include a manifest comment at the top of each artifact file:
  ```
  // Generated by: spec-<skill-name> skill
  // Source spec: .sdd/specs/<NNN>-<idea-name>.spec.md, Section <N>
  // Target language: <language>
  // DO NOT EDIT MANUALLY -- regenerated on spec revision
  ```

---

### 4.3 Requirements Skill (spec-requirements) -- Section 4

- **FR-029**: The spec-requirements skill SHALL produce Section 4 (Functional Requirements) of the spec, organized by feature area. For each requirement:
  1. A unique FR-XXX identifier
  2. A SHALL or SHALL NOT obligation statement
  3. Preconditions (if non-trivial)
  4. Postconditions (expected state after the requirement is satisfied)
  5. Error behavior (what happens when the happy path fails)
  6. `[NEEDS CLARIFICATION]` markers for unresolved decisions

- **FR-030**: The skill SHALL also produce Section 10 (Non-Functional Requirements) covering performance, security overview, scalability, accessibility, and observability with measurable targets.

- **FR-031**: The skill SHALL produce Section 12 (Constraints & Assumptions) and Section 13 (Out of Scope).

- **FR-032**: The skill SHALL produce an Implementation Contract subsection for each feature area in Section 4 that defines exact inputs, outputs, and error behaviors.

#### Implementation Contract -- spec-requirements

**Inputs**: Accumulator (sections 1-3), brief, research summary, patterns.
**Outputs**: Sections 4, 10, 12, 13 appended to accumulator.
**Artifacts**: None (requirements are prose).

---

### 4.4 User Stories Skill (spec-user-stories) -- Section 5, 6

- **FR-033**: The spec-user-stories skill SHALL produce Section 5 (User Stories) with priority-ordered stories. Each story SHALL include:
  1. Unique US-XX identifier
  2. Priority (P1, P2, P3) with rationale
  3. "As a / I want / so that" format
  4. Independent Test statement (how to verify in isolation)
  5. Acceptance Scenarios in Given/When/Then format (at least one happy path, one error path)
  6. Edge Cases subsection

- **FR-034**: The skill SHALL produce Section 6 (User Flows) with numbered step-by-step flows for each primary flow, including actor actions, system responses, and branching conditions.

- **FR-035**: The skill SHALL cross-reference user stories against FRs from the accumulator (Section 4) to ensure every US maps to at least one FR and every FR is covered by at least one US. Missing mappings SHALL be added as notes for the traceability skill.

#### Implementation Contract -- spec-user-stories

**Inputs**: Accumulator (sections 1-4, 10, 12, 13), brief, research summary, patterns.
**Outputs**: Sections 5, 6 appended to accumulator.
**Artifacts**: None (stories are prose).

---

### 4.5 Data Model Skill (spec-data-model) -- Section 7

- **FR-036**: The spec-data-model skill SHALL produce Section 7 (Data Model) defining every entity with:
  1. Entity name
  2. Fields: name, type (including nullability), constraints (required, unique, max length, format regex, enum values, min/max), default value
  3. Relationships to other entities with cardinality (1:1, 1:N, N:M)
  4. Validation rules (beyond type constraints: cross-field validation, business rules)
  5. State machine definition (if the entity has a status/state field): valid states, valid transitions, guards, side effects

- **FR-037**: The skill SHALL produce companion artifact files in the artifacts directory:
  1. `data-models.<ext>`: Entity definitions as typed classes/interfaces/structs in the target language
  2. `state-machines.<ext>`: State transition enums and validation functions (if any entity has state fields)
  - Each entity in the prose Section 7 SHALL have a corresponding definition in the artifact file.

- **FR-038**: Field names, types, constraints, and defaults in the companion artifact SHALL match the prose Section 7 exactly. No interpretation or renaming.

#### Implementation Contract -- spec-data-model

**Inputs**: Accumulator (sections 1-6), brief, research summary, patterns, target language.
**Outputs**: Section 7 appended to accumulator.
**Artifacts**: `data-models.<ext>`, `state-machines.<ext>` (if applicable).

---

### 4.6 API Design Skill (spec-api-design) -- Section 8

- **FR-039**: The spec-api-design skill SHALL produce Section 8 (API / Interface Design) defining every endpoint or interface:
  1. Method + path (or function signature for libraries/CLIs)
  2. Purpose (one line)
  3. Request: parameters, body schema with all fields typed, validation rules
  4. Response: success schema with all fields typed, every applicable error code (400, 401, 403, 404, 409, 422, 500) with meaning and response body
  5. Auth requirements
  6. Rate limits (if applicable)

- **FR-040**: The skill SHALL produce companion artifact files:
  1. `api-contracts.<ext>`: Request/response type definitions in the target language
  2. `error-catalog.<ext>`: Error code constants/enums with HTTP status codes and message templates
  - Every endpoint's request and response types SHALL have corresponding definitions in the artifact file.

- **FR-041**: Request/response field names and types in the companion artifact SHALL match the prose Section 8 exactly. Error codes in the error catalog SHALL match Section 4 error behaviors.

- **FR-042**: The skill SHALL cross-reference API endpoints against the data model (Section 7 from the accumulator) to verify that response schemas use the same entity fields. Mismatches SHALL be flagged with `[CROSS-REF ISSUE]`.

#### Implementation Contract -- spec-api-design

**Inputs**: Accumulator (sections 1-7), brief, research summary, patterns, target language.
**Outputs**: Section 8 appended to accumulator.
**Artifacts**: `api-contracts.<ext>`, `error-catalog.<ext>`.

---

### 4.7 Architecture Skill (spec-architecture) -- Section 9

- **FR-043**: The spec-architecture skill SHALL produce Section 9 (Architecture) with:
  1. Section 9.1 System Design: high-level component description and interaction diagram (Mermaid or prose)
  2. Section 9.2 Technology Stack: table with Layer, Technology, Version, Rationale columns
  3. Section 9.3 Directory & Module Structure: proposed folder structure with one-line descriptions
  4. Section 9.4 Key Design Decisions: for each decision, the decision, rationale, alternatives considered, consequences, and source references
  5. Section 9.5 External Integrations: for each external system, purpose, auth method, key operations, timeout/retry/fallback strategy

- **FR-044**: The skill SHALL produce companion artifact files:
  1. `config-schema.<ext>`: Configuration/environment variable definitions with types, defaults, validation, and descriptions
  - Every env var referenced in the architecture SHALL have a definition in the artifact file.

- **FR-045**: The skill SHALL verify that the proposed directory structure accommodates all entities from the data model (Section 7) and all endpoints from the API design (Section 8).

- **FR-046**: The skill SHALL specify virtual environment usage: Python projects MUST use venv/poetry/conda; Node projects MUST use local node_modules; never global package installation.

#### Implementation Contract -- spec-architecture

**Inputs**: Accumulator (sections 1-8), brief, research summary, patterns, target language.
**Outputs**: Section 9 appended to accumulator.
**Artifacts**: `config-schema.<ext>`.

---

### 4.8 Security Skill (spec-security) -- Section 10.2 (expansion)

- **FR-047**: The spec-security skill SHALL expand Section 10.2 (Security) with:
  1. Authentication mechanism details (protocol, token format, expiry, refresh)
  2. Authorization model (RBAC/ABAC) with roles and permissions matrix
  3. Data sensitivity classification per entity (public, internal, confidential, restricted)
  4. OWASP Top 10 mitigations required for this system's specific threat surface
  5. Per-component security requirements (which components handle auth, which handle sensitive data)
  6. Input validation strategy (allow-list vs deny-list, centralized vs per-endpoint)
  7. Secrets management approach (env vars, vault, key management)

- **FR-048**: The skill SHALL cross-reference security requirements against the data model (Section 7) to ensure every entity with sensitive fields has appropriate handling rules (encryption at rest, masking in logs, access control).

- **FR-049**: The skill SHALL use web research to verify security recommendations against current OWASP guidelines and framework-specific security documentation.

#### Implementation Contract -- spec-security

**Inputs**: Accumulator (sections 1-9), brief, research summary, patterns.
**Outputs**: Section 10.2 expanded in accumulator.
**Artifacts**: None (security requirements are prose with references).

---

### 4.9 Test Strategy Skill (spec-test-strategy) -- Section 11

- **FR-050**: The spec-test-strategy skill SHALL produce Section 11 (Test Requirements) with:
  1. Section 11.1 Unit Tests: modules requiring coverage, minimum threshold (80% code, 90% branch), specific edge cases
  2. Section 11.2 BDD / Acceptance Tests: Gherkin scenarios for every acceptance criterion from Section 5 user stories -- 1:1 mapping required
  3. Section 11.3 Integration Tests: component boundaries, external dependency mocking strategy, data setup/teardown
  4. Section 11.4 End-to-End Tests: critical user journeys, target environments, tools
  5. Section 11.5 Performance Tests: scenarios, thresholds
  6. Section 11.6 Security Tests: OWASP checks, auth/authz test cases

- **FR-051**: Every acceptance scenario from Section 5 SHALL have a corresponding Gherkin scenario in Section 11.2. The skill SHALL report any missing mappings.

- **FR-052**: The skill SHALL emphasize BDD/TDD: tests derive from acceptance scenarios, not from implementation. Every acceptance scenario SHALL have a corresponding test.

#### Implementation Contract -- spec-test-strategy

**Inputs**: Accumulator (sections 1-10), brief, research summary, patterns.
**Outputs**: Section 11 appended to accumulator.
**Artifacts**: None (test strategy is prose with Gherkin).

---

### 4.10 Traceability Skill (spec-traceability) -- Sections 14, 15, 16, 17, 18

- **FR-053**: The spec-traceability skill SHALL produce:
  1. Section 14 (Open Questions): unresolved decisions with impact and owner
  2. Section 15 (Glossary): key terms, acronyms, domain concepts
  3. Section 16 (Traceability Matrix): table mapping every FR -> US -> Acceptance Scenario -> Test Type -> Test Section Ref
  4. Section 17 (Technical References): sources consulted, grouped by topic, with URLs and dates
  5. Section 18 (Version History): initial version entry

- **FR-054**: The traceability matrix SHALL have no empty cells. For every FR:
  - At least one US mapping
  - At least one acceptance scenario
  - At least one test type (unit, BDD, integration, E2E, performance, security)
  - At least one test section reference (11.1, 11.2, etc.)
  - If any cell is empty, the skill SHALL flag it and attempt to fill it by cross-referencing other sections.

- **FR-055**: The skill SHALL validate the entire spec for completeness:
  1. Every FR-XXX referenced in the traceability matrix exists in Section 4
  2. Every US-XX referenced exists in Section 5
  3. Every Gherkin scenario in Section 11.2 maps to an acceptance scenario in Section 5
  4. No orphan FRs (FRs not referenced by any US)
  5. No orphan USes (USes not referencing any FR)
  - Validation failures SHALL be reported as `[TRACEABILITY GAP: <description>]` markers.

#### Implementation Contract -- spec-traceability

**Inputs**: Accumulator (sections 1-11), brief, research summary.
**Outputs**: Sections 14, 15, 16, 17, 18 appended to accumulator.
**Artifacts**: None.

---

## 5. User Stories

### US-01 -- Generate a Deep Specification (Priority: P1) MVP

**As a** Human Developer, **I want** the Spec Architect to produce a comprehensive specification with language-specific companion artifacts, **so that** downstream agents can implement faithfully without interpretation.

**Why P1**: This is the core value proposition -- deeper specs reduce pipeline drift.

**Independent Test**: Provide a brainstorming brief. Invoke the Spec Architect. Verify: a spec with all 16+ sections populated, companion artifact files in `.sdd/specs/artifacts/`, all FRs use SHALL, all entities have typed fields, all endpoints have error codes, traceability matrix has no gaps.

**Acceptance Scenarios**:
1. **Given** a brainstorming brief for a REST API project, **When** the Spec Architect completes, **Then** the spec file contains all 16+ sections with no `[NEEDS CLARIFICATION]` markers, and the artifacts directory contains `data-models.ts`, `api-contracts.ts`, `error-catalog.ts`, and `config-schema.ts`.
2. **Given** a brief for a Python CLI tool, **When** the Spec Architect completes, **Then** companion artifacts are in Python (`.py` files) with dataclass definitions and Protocol classes.
3. **Given** a brief with ambiguous requirements, **When** the coordinator runs gap analysis, **Then** the user is asked focused questions (max 3 per turn) before any skill is dispatched.

---

### US-02 -- Sequential Skill Dispatch with Context Forwarding (Priority: P1) MVP

**As the** Spec Architect coordinator, **I want** each skill to read the accumulator file before writing its section, **so that** later sections are consistent with earlier ones.

**Why P1**: Without context forwarding, sections contradict each other.

**Independent Test**: Dispatch spec-data-model followed by spec-api-design. Verify: API response schemas reference entity fields defined by the data model skill, using the same names and types.

**Acceptance Scenarios**:
1. **Given** spec-data-model defines a `User` entity with fields `id: UUID`, `email: string`, `role: enum(admin, user)`, **When** spec-api-design runs, **Then** the GET /users/:id response schema references the same field names and types.
2. **Given** spec-requirements defines FR-012 with a specific error behavior, **When** spec-user-stories runs, **Then** at least one acceptance scenario tests that error behavior.
3. **Given** a skill discovers an inconsistency with a prior section, **When** it writes its section, **Then** it adds a `[CROSS-REF ISSUE]` marker without modifying the prior section.

---

### US-03 -- Companion Artifact Generation (Priority: P1) MVP

**As the** Planner and Coder agents, **I want** language-specific companion artifacts alongside the prose spec, **so that** I can use precise technical definitions without interpreting prose.

**Why P1**: Companion artifacts are the mechanism that eliminates interpretation-based drift.

**Independent Test**: After spec generation, verify: each companion artifact file contains valid syntax in the target language, field names match the prose spec exactly, and a manifest comment references the source spec section.

**Acceptance Scenarios**:
1. **Given** a spec with 5 entities in the data model, **When** the data model skill completes, **Then** `data-models.ts` contains 5 interface/type definitions with all fields typed.
2. **Given** a spec with 10 API endpoints, **When** the API design skill completes, **Then** `api-contracts.ts` contains 10 request types and 10 response types.
3. **Given** an entity with a status field and 4 valid states, **When** the data model skill completes, **Then** `state-machines.ts` contains a state enum and a transition validation function.

---

### US-04 -- Post-Completion Validation (Priority: P1) MVP

**As a** Human Developer, **I want** the Spec Architect to validate its own output before presenting it to me, **so that** I review a complete, consistent spec rather than catching obvious gaps.

**Why P1**: Self-validation prevents incomplete specs from reaching the human gate.

**Independent Test**: Introduce a deliberate gap (e.g., an FR without error behavior). Verify: the coordinator catches it during post-completion validation and either fixes it or asks the user.

**Acceptance Scenarios**:
1. **Given** a spec with an FR that uses "should" instead of "SHALL", **When** post-completion validation runs, **Then** the coordinator replaces it with "SHALL".
2. **Given** a traceability matrix with an empty cell, **When** validation runs, **Then** the coordinator fills it or flags it for user review.
3. **Given** companion artifacts with a field name that differs from the prose spec, **When** validation runs, **Then** the coordinator resolves the inconsistency.

---

### US-05 -- Dynamic Spec Skill Discovery (Priority: P2)

**As a** system maintainer, **I want** to add a new spec skill by creating a `.github/skills/spec-<name>/SKILL.md` file, **so that** the Spec Architect dispatches it automatically.

**Why P2**: Extensibility enables future spec dimensions without coordinator changes.

**Independent Test**: Add a `.github/skills/spec-compliance/SKILL.md` file. Run the Spec Architect. Verify: the new skill is discovered and dispatched after all known skills.

**Acceptance Scenarios**:
1. **Given** 8 spec skills installed, **When** a 9th skill `spec-compliance` is added, **Then** 9 skills are dispatched.
2. **Given** a skill `spec-foo` is removed, **When** the Spec Architect runs, **Then** remaining skills are dispatched without error.

---

### Edge Cases

- What happens when the brief is extremely vague? The coordinator enters an extended gap analysis loop (FR-006) with the user until critical gaps are resolved.
- What happens when two skills define conflicting requirements? The coordinator detects `[CROSS-REF ISSUE]` markers during post-completion validation (FR-017, FR-018) and resolves them.
- What happens when the target language is not specified? The architecture skill defaults to TypeScript and notes the assumption (FR-015).
- What happens when a skill's section is too large for a single edit? Skills write in blocks, reading the accumulator between writes to maintain consistency.

---

## 6. User Flows

### 6.1 Full Specification Generation Flow

1. User or Orchestrator invokes the Spec Architect.
2. Coordinator lists briefs in `.sdd/ideas/` and asks user to select one (or confirms if only one exists).
3. Coordinator reads selected brief in full.
4. Coordinator invokes research subagent for workspace discovery.
5. Coordinator conducts web research (competitors, tech versions, OWASP, standards).
6. Coordinator identifies gaps and tracks them.
7. Coordinator asks user questions (max 3 per turn) to resolve gaps. Repeats until critical gaps resolved.
8. Coordinator reads spec-patterns.md for active patterns (if exists).
9. Coordinator creates accumulator file with header + sections 1-3.
10. Coordinator creates companion artifacts directory.
11. Coordinator discovers spec skills via glob scan.
12. For each skill in canonical order:
    a. Coordinator invokes `runSubagent` with skill prompt including accumulator path and context.
    b. Skill reads SKILL.md, reads current accumulator, reads brief.
    c. Skill writes its section(s) to the accumulator file.
    d. Skill writes companion artifacts (if applicable).
    e. Coordinator waits for skill to return.
    f. If skill fails, coordinator halts with error.
13. Coordinator runs post-completion validation (FR-017).
14. Coordinator verifies artifact consistency (FR-018).
15. Coordinator resolves any `[CROSS-REF ISSUE]` markers.
16. Coordinator presents completed spec to user.
17. User reviews and provides feedback:
    a. Approval: coordinator sets status to "Validated", commits all files, stops.
    b. Changes: coordinator revises (re-dispatch affected skills or inline edit), re-validates, re-presents.
    c. Questions: coordinator clarifies or asks follow-up questions.
    d. New requirements: coordinator loops back to step 4.

### 6.2 Spec Revision Flow

1. User or Planner/Reviewer requests spec revision with specific changes.
2. Coordinator reads existing spec and identifies affected sections.
3. Coordinator re-dispatches only the skills responsible for affected sections (plus dependent skills).
4. Re-dispatched skills read the full accumulator and rewrite their sections.
5. Coordinator re-validates and re-checks artifact consistency.
6. Coordinator presents revised spec.
7. Coordinator commits with revision message.

---

## 7. Data Model

### 7.1 Accumulator File (Spec)

**Path**: `.sdd/specs/<NNN>-<idea-name>.spec.md`

The accumulator file IS the spec itself. Skills append to it sequentially:

| Section | Written By | Content |
|---------|-----------|---------|
| Header + 1-3 | Coordinator | Overview, goals, users |
| 4, 10, 12, 13 | spec-requirements | FRs, NFRs, constraints, out of scope |
| 5, 6 | spec-user-stories | User stories, flows |
| 7 | spec-data-model | Entities, fields, relationships |
| 8 | spec-api-design | Endpoints, schemas, error codes |
| 9 | spec-architecture | System design, tech stack, structure |
| 10.2 (expanded) | spec-security | Security details |
| 11 | spec-test-strategy | Test requirements, BDD scenarios |
| 14, 15, 16, 17, 18 | spec-traceability | Open questions, glossary, matrix, refs, history |

### 7.2 Companion Artifact Files

**Path**: `.sdd/specs/artifacts/<NNN>-<idea-name>/`

| File | Produced By | Content |
|------|-----------|---------|
| `data-models.<ext>` | spec-data-model | Entity type definitions |
| `state-machines.<ext>` | spec-data-model | State enums and transition validators |
| `api-contracts.<ext>` | spec-api-design | Request/response type definitions |
| `error-catalog.<ext>` | spec-api-design | Error code constants |
| `interfaces.<ext>` | spec-api-design | Function/method signatures |
| `config-schema.<ext>` | spec-architecture | Configuration definitions |

### 7.3 Spec Architect Agent File

**Path**: `.github/agents/spec-architect.agent.md`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `name` | string | `"2. Spec Architect"` | Agent display name |
| `description` | string | Trigger keywords for spec writing | When invoked |
| `model` | string | `Claude Opus 4.6 (copilot)` | LLM model |
| `tools` | array(string) | Must include: `agent/runSubagent`, file ops, search, web, todo, askQuestions | Required tools |
| `handoffs` | array(object) | To Planner, to Ideation | Available targets |

### 7.4 Spec Skill File

**Path**: `.github/skills/spec-<name>/SKILL.md`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `name` | string | `spec-<name>` pattern | Skill identifier |
| `description` | string | 1-500 characters | Purpose and scope |

Skill body contains: purpose statement, section template, quality guidelines (SHALL language, error behavior requirements, completeness criteria), companion artifact format instructions (if applicable).

---

## 8. API / Interface Design

### 8.1 Coordinator Invocation Interface

The coordinator is invoked as a VS Code chat agent.

**Invocation methods**:
1. Direct: user selects "Spec Architect" agent mode
2. Handoff: Orchestrator delegates when a brief exists without a spec
3. Handoff: Reviewer requests spec revision

**Response**: Markdown-formatted spec presented in the VS Code chat panel, plus file modifications (spec file, artifact files, commit).

### 8.2 Skill Subagent Prompt Template

```
Write Section <section_numbers> of the specification.

1. Read the skill instructions at: <skill_path>
2. Read the current spec state at: <accumulator_path>
3. Read the source brief at: <brief_path>
4. Research context: <research_summary>
5. Active patterns to avoid: <patterns>
6. Target language for artifacts: <target_language>
7. Artifacts directory: <artifacts_dir>

Write your section(s) to the spec file at <accumulator_path> by APPENDING after the existing content.
If this skill produces companion artifacts, write them to <artifacts_dir>.

Rules:
- Read the existing spec content to maintain consistency with prior sections
- Use SHALL/SHALL NOT for all requirements (never "should")
- Include error behavior for every FR
- Include an Implementation Contract subsection per feature area
- Use [CROSS-REF ISSUE: description] if you find inconsistencies with prior sections
- Do NOT modify any existing sections -- only append your assigned sections
- Follow the quality guidelines in your SKILL.md
```

### 8.3 Handoff Prompt Templates

**Decompose into Work Packages** (to Planner):
```
Specification <spec_path> has been validated and approved.
Companion artifacts are at: <artifacts_dir>
Please decompose into work packages with contracts.
```

**Return to Ideation** (to Ideation):
```
The specification process has identified fundamental issues with the brief.
Issues: <list>
Please revise the ideation brief.
```

---

## 9. Architecture

### 9.1 System Design

The system consists of two component types:

1. **Spec Architect Coordinator** (`.github/agents/spec-architect.agent.md`): Lightweight dispatcher that owns the spec lifecycle. Handles: brief selection, research, gap analysis, accumulator initialization, skill discovery, sequential dispatch with context forwarding, post-completion validation, artifact consistency checks, user interaction, and committing.

2. **Spec Skill Files** (`.github/skills/spec-*/SKILL.md`): Self-contained section writers loaded by subagents. Each skill writes one or more spec sections to the shared accumulator file and optionally produces companion artifact files.

**Interaction pattern**:
```
User/Orchestrator
       |
       v
Spec Architect Coordinator
       |
       |--> Research (subagent + web)
       |--> Gap analysis (user Q&A)
       |--> Initialize accumulator (sections 1-3)
       |--> Create artifacts directory
       |--> Discover skills (scan .github/skills/spec-*/)
       |
       |--> runSubagent(spec-requirements)     --> writes sections 4,10,12,13
       |--> runSubagent(spec-user-stories)     --> writes sections 5,6
       |--> runSubagent(spec-data-model)       --> writes section 7 + artifacts
       |--> runSubagent(spec-api-design)       --> writes section 8 + artifacts
       |--> runSubagent(spec-architecture)     --> writes section 9 + artifacts
       |--> runSubagent(spec-security)         --> expands section 10.2
       |--> runSubagent(spec-test-strategy)    --> writes section 11
       |--> runSubagent(spec-traceability)     --> writes sections 14-18
       |
       |--> Post-completion validation
       |--> Artifact consistency check
       |--> Resolve CROSS-REF markers
       |--> Present to user
       |--> Commit on approval
```

### 9.2 Technology Stack

| Layer | Technology | Version | Rationale |
|-------|-----------|---------|-----------|
| Agent framework | VS Code Copilot Chat agents | Current | Existing SDD infrastructure; provides runSubagent, file ops, search |
| Skill framework | VS Code Copilot Chat skills | Current | Proven in Reviewer V2; skills discovered via glob patterns |
| Data format | Markdown + YAML frontmatter | N/A | Consistent with all SDD artifacts; human-readable, diff-friendly |
| Companion artifacts | Target language source files | Varies | Eliminates interpretation; Coder can copy verbatim |
| Version control | Git | Current | Explicit git add + git commit for all artifacts |

### 9.3 Directory & Module Structure

```
.github/
  agents/
    spec-architect.agent.md           # MODIFIED: refactored to coordinator pattern
  skills/
    spec-requirements/SKILL.md        # NEW: FR, NFR, constraints, out of scope
    spec-user-stories/SKILL.md        # NEW: user stories, flows
    spec-data-model/SKILL.md          # NEW: entities, fields, artifacts
    spec-api-design/SKILL.md          # NEW: endpoints, schemas, error catalog
    spec-architecture/SKILL.md        # NEW: system design, tech stack, config
    spec-security/SKILL.md            # NEW: security requirements expansion
    spec-test-strategy/SKILL.md       # NEW: test requirements, BDD scenarios
    spec-traceability/SKILL.md        # NEW: traceability matrix, glossary, refs

.sdd/
  specs/
    <NNN>-<idea-name>.spec.md         # Accumulator file (the spec)
    artifacts/
      <NNN>-<idea-name>/
        data-models.<ext>
        state-machines.<ext>
        api-contracts.<ext>
        error-catalog.<ext>
        interfaces.<ext>
        config-schema.<ext>
  reviews/
    spec-patterns.md                  # NEW: domain-specific patterns for specs
```

### 9.4 Key Design Decisions

**Decision 1: Shared accumulator file for context forwarding**
- **Rationale**: Skills read the same spec file that prior skills have written to, giving them full context without coordinator prompt injection. Natural and simple.
- **Alternatives considered**: Coordinator passes prior skill output in prompt (context bloat), separate files per skill then merge (consistency risk).
- **Consequences**: Skills must append only, never modify prior sections. CROSS-REF markers handle inconsistencies gracefully.

**Decision 2: Skills halt the pipeline on failure (unlike Reviewer)**
- **Rationale**: Spec skills are sequential and dependent. Section 8 (API Design) cannot be written without Section 7 (Data Model). In the Reviewer, skills are independent, so failure tolerance makes sense.
- **Alternatives considered**: Continue with placeholder sections (produces invalid spec), skip the section (creates traceability gaps).
- **Consequences**: Any skill failure requires debugging before the spec can be completed.

**Decision 3: Companion artifacts in target language**
- **Rationale**: User explicitly chose language-specific code over structured markdown or pseudo-code. Eliminates the translation step that causes drift.
- **Alternatives considered**: OpenAPI YAML, structured markdown tables, language-agnostic pseudo-code.
- **Consequences**: Spec Architect must know the target language. Default to TypeScript if not specified.

**Decision 4: 80% technical design in spec, 20% in Planner**
- **Rationale**: User wants the spec to be the authoritative source of technical truth. Planner fills implementation-specific gaps (task-level contracts, WP-scoped refinements).
- **Alternatives considered**: 50/50 split, Planner owns all technical design.
- **Consequences**: Spec becomes more substantial; Planner's contract generation is additive, not from scratch.

### 9.5 External Integrations

**Web research (via fetch_webpage)**:
- Purpose: Technology evaluation, OWASP references, competitor analysis.
- Authentication: None (public web).
- Key operations: Fetch official docs, GitHub repos, security guidelines.
- Failure handling: If web fetch fails, note assumption and continue with available information.

---

## 10. Non-Functional Requirements

### 10.1 Performance

- **NFR-001**: A full spec generation (8 skills) SHALL complete within 45 minutes for a medium-complexity project (10-20 entities, 20-40 endpoints).
- **NFR-002**: The coordinator's own processing (research, gap analysis, validation, commit) SHALL complete in under 10 minutes. The bulk of time is in skill subagent execution.
- **NFR-003**: Spec revision (re-dispatching 1-3 skills) SHALL complete within 20 minutes.

### 10.2 Security

- **NFR-004**: The coordinator and skills SHALL NOT store credentials, tokens, or API keys in the spec or companion artifacts.
- **NFR-005**: Companion artifact files SHALL NOT contain executable code that performs I/O, network access, or file system operations. They contain type definitions, interfaces, and schemas only.
- **NFR-006**: Web research URLs SHALL only target well-known resources (official docs, OWASP, RFC).

### 10.3 Scalability & Availability

- Local workspace only. No availability or scaling requirements.
- **NFR-007**: The system SHALL handle specs up to 3000 lines without degraded quality.
- **NFR-008**: The system SHALL handle up to 10 companion artifact files per spec without performance issues.

### 10.4 Accessibility

- Not applicable. Output is markdown and source code files in VS Code.

### 10.5 Observability

- **Logging**: The spec's Version History (Section 18) serves as the audit trail.
- **Metrics**: Skill count dispatched, time per skill (observable from terminal output).
- **Alerting**: Validation failures are surfaced to the user directly.

---

## 11. Test Requirements

### 11.1 Unit Tests

Not applicable. The "units" are agent and skill markdown files.

Validation SHALL verify:
- Coordinator agent file has valid YAML frontmatter + valid markdown.
- Each skill file has valid YAML frontmatter + valid markdown.
- Companion artifact files contain valid syntax in the target language.

### 11.2 BDD / Acceptance Tests

```gherkin
Feature: Spec Architect V2 - Full Specification Generation

  Scenario: Generate a complete spec from a brainstorming brief
    Given a brief "002-pipeline-v2.md" exists in .sdd/ideas/
    And 8 spec skills are installed
    When the Spec Architect is invoked and the user selects the brief
    Then all 8 skills are dispatched sequentially
    And the spec file contains all 16+ sections
    And the artifacts directory contains at least 4 companion files
    And no [NEEDS CLARIFICATION] markers remain
    And no [CROSS-REF ISSUE] markers remain
    And a git commit is created

  Scenario: Context forwarding between skills
    Given spec-data-model defines a User entity with field "email: string"
    When spec-api-design runs and reads the accumulator
    Then the POST /users request schema includes field "email: string"
    And field names match exactly between prose and artifact

  Scenario: Post-completion validation catches ambiguity
    Given a skill writes an FR using "should" instead of "SHALL"
    When post-completion validation runs
    Then the coordinator replaces "should" with "SHALL"

  Scenario: Companion artifact consistency check
    Given data-models.ts defines User with field "createdAt: Date"
    And the prose Section 7 defines User with field "created_at: datetime"
    When artifact consistency validation runs
    Then the coordinator detects the naming mismatch and resolves it

  Scenario: Skill failure halts pipeline
    Given spec-data-model encounters an unrecoverable error
    When the coordinator detects the failure
    Then it halts immediately and reports the error
    And no subsequent skills are dispatched

  Scenario: Gap analysis with user
    Given a brief with unspecified authentication method
    When the coordinator runs gap analysis
    Then it asks the user (max 3 questions per turn) to clarify
    And does not dispatch skills until the gap is resolved

  Scenario: Dynamic skill discovery
    Given 8 spec skills exist
    When a 9th skill "spec-compliance" is added
    And the Spec Architect runs
    Then 9 skills are dispatched

  Scenario: Spec revision re-dispatches affected skills
    Given an approved spec where Section 7 (Data Model) needs changes
    When the coordinator receives revision request
    Then it re-dispatches spec-data-model and spec-api-design (dependent)
    And re-validates after re-dispatch
```

### 11.3 Integration Tests

- Verify the full pipeline: Ideation brief -> Spec Architect V2 -> Spec output consumed by Planner V2.
- Verify companion artifacts are parseable by the Coder agent.

### 11.4 End-to-End Tests

- Manual test: provide a real brainstorming brief (e.g., the 002 brief from this project) and run the Spec Architect V2 through completion. Verify all outputs.

### 11.5 Performance Tests

- Time the full 8-skill dispatch on a medium-complexity brief. Target: under 45 minutes.

### 11.6 Security Tests

- Verify no credentials or secrets appear in spec or artifact files.
- Verify artifact files contain only type definitions, no executable I/O code.

---

## 12. Constraints & Assumptions

### Constraints

- Must operate within the VS Code Copilot Chat agent framework (.agent.md, SKILL.md, runSubagent).
- Skills execute sequentially (framework limitation for subagents).
- Context window limits may require skills to read only their relevant sections of the accumulator, not the entire file, for very large specs.
- All artifacts are local files; no external database or service.

### Assumptions

1. The runSubagent tool supports file read/write operations within the subagent context.
2. Sequential skill dispatch with shared accumulator file produces consistent output (validated by Reviewer V2 pattern).
3. LLMs can reliably generate valid syntax in the target language for companion artifacts.
4. 8 skills is sufficient for comprehensive spec coverage; additional skills can be added via dynamic discovery.
5. The 800-line block approach for large sections prevents context window exhaustion.

---

## 13. Out of Scope

- **Code implementation**: The Spec Architect produces specifications and artifacts, not executable code.
- **Plan decomposition**: Work package generation is the Planner's responsibility.
- **Review of specifications**: The review-spec-completeness skill (Spec 005) handles validation; the Spec Architect does not review other specs.
- **Runtime deployment**: No deployment or infrastructure concerns.
- **Multi-language artifacts**: Each spec targets one language. Multi-language projects need separate artifact sets (future enhancement).

---

## 14. Open Questions

None remaining. All questions from the brainstorming brief have been resolved:
1. Skill ordering: Requirements -> User Stories -> Data Model -> API -> Architecture -> Security -> Tests -> Traceability (resolved in FR-010).
2. Contract versioning: Planner regenerates only affected contracts (resolved in brief, impacts Spec 003).
3. Docs Agent trigger: After every WP approval (resolved in brief, impacts Spec 007).

---

## 15. Glossary

- **Accumulator file**: The spec markdown file that skills append to sequentially. Each skill reads the current state before writing.
- **Companion artifact**: A language-specific code file (interfaces, schemas, enums) produced alongside the prose spec. Stored in `.sdd/specs/artifacts/`.
- **Context forwarding**: The pattern where a later skill reads content written by an earlier skill via the shared accumulator file.
- **CROSS-REF ISSUE**: An inline marker a skill adds when it detects an inconsistency with a prior section it cannot modify.
- **Gap analysis**: The process of identifying and resolving missing information from the brief before spec writing begins.
- **SHALL/SHALL NOT**: Mandatory obligation language (RFC 2119). Every functional requirement uses this.
- **Skill dispatch**: Invoking a skill as a subagent via `runSubagent` with a structured prompt.
- **Spec skill**: A SKILL.md file under `.github/skills/spec-*/` that writes one or more spec sections.

---

## 16. Traceability Matrix

| FR ID | Requirement Summary | User Story | Acceptance Scenario | Test Type | Test Section Ref |
|-------|-------------------|------------|--------------------|-----------|----|
| FR-001 | Brief selection from .sdd/ideas/ | US-01 | Scenario 1 | BDD | 11.2 |
| FR-002 | Read brief in full | US-01 | Scenario 1 | BDD | 11.2 |
| FR-003 | Workspace research subagent | US-01 | Scenario 1 | BDD | 11.2 |
| FR-004 | Mandatory web research | US-01 | Scenario 1 | BDD | 11.2 |
| FR-005 | Gap identification and categorization | US-01 | Scenario 3 | BDD | 11.2 |
| FR-006 | User questions for gap resolution | US-01 | Scenario 3, BDD Scenario 6 | BDD | 11.2 |
| FR-007 | Accumulator file initialization | US-01 | Scenario 1 | BDD | 11.2 |
| FR-008 | Artifacts directory creation | US-03 | Scenario 1 | BDD | 11.2 |
| FR-009 | Dynamic skill discovery | US-05 | Scenario 1, 2, BDD Scenario 7 | BDD | 11.2 |
| FR-010 | Deterministic skill ordering | US-02 | Scenario 1 | BDD | 11.2 |
| FR-011 | Skill dispatch via runSubagent | US-02 | Scenario 1 | BDD | 11.2 |
| FR-012 | Sequential blocking execution | US-02 | Scenario 1, BDD Scenario 5 | BDD | 11.2 |
| FR-013 | Accumulator read before write | US-02 | Scenario 1, BDD Scenario 2 | BDD | 11.2 |
| FR-014 | Companion artifact generation | US-03 | Scenario 1, 2, 3 | BDD | 11.2 |
| FR-015 | Target language for artifacts | US-03 | Scenario 2 | BDD | 11.2 |
| FR-016 | Artifact file naming | US-03 | Scenario 1 | BDD | 11.2 |
| FR-017 | Post-completion validation | US-04 | Scenario 1, 2, BDD Scenario 3 | BDD | 11.2 |
| FR-018 | Artifact consistency check | US-04 | Scenario 3, BDD Scenario 4 | BDD | 11.2 |
| FR-019 | Patterns consumption | US-01 | Scenario 1 | BDD | 11.2 |
| FR-020 | Presentation to user | US-04 | Scenario 1 | BDD | 11.2 |
| FR-021 | User feedback handling | US-01, US-04 | Scenario 3, BDD Scenario 8 | BDD | 11.2 |
| FR-022 | Commit spec and artifacts | US-01 | Scenario 1 | BDD | 11.2 |
| FR-023 | Skill input contract | US-02 | Scenario 1 | BDD | 11.2 |
| FR-024 | Skill execution sequence | US-02 | Scenario 1 | BDD | 11.2 |
| FR-025 | Skill output format | US-02 | Scenario 1 | BDD | 11.2 |
| FR-026 | No modification of prior sections | US-02 | Scenario 3 | BDD | 11.2 |
| FR-027 | No modification of coordinator sections | US-02 | Scenario 1 | BDD | 11.2 |
| FR-028 | Artifact manifest comments | US-03 | Scenario 1 | BDD | 11.2 |
| FR-029 | Requirements skill: FR section | US-01 | Scenario 1 | BDD | 11.2 |
| FR-030 | Requirements skill: NFR section | US-01 | Scenario 1 | BDD | 11.2 |
| FR-033 | User stories skill: stories | US-01, US-02 | Scenario 2 | BDD | 11.2 |
| FR-036 | Data model skill: entities | US-03 | Scenario 1, BDD Scenario 2 | BDD | 11.2 |
| FR-037 | Data model skill: artifacts | US-03 | Scenario 1, 3 | BDD | 11.2 |
| FR-039 | API design skill: endpoints | US-03 | Scenario 2 | BDD | 11.2 |
| FR-040 | API design skill: artifacts | US-03 | Scenario 2 | BDD | 11.2 |
| FR-043 | Architecture skill: system design | US-01 | Scenario 1 | BDD | 11.2 |
| FR-047 | Security skill: expansion | US-01 | Scenario 1 | BDD | 11.2 |
| FR-050 | Test strategy skill: sections | US-01 | Scenario 1 | BDD | 11.2 |
| FR-053 | Traceability skill: matrix | US-01, US-04 | Scenario 2 | BDD | 11.2 |
| FR-054 | Traceability: no empty cells | US-04 | Scenario 2 | BDD | 11.2 |
| FR-055 | Traceability: validation | US-04 | Scenario 2 | BDD | 11.2 |

---

## 17. Technical References

### Architecture & Patterns
- arc42 Template Overview, https://arc42.org/overview, consulted 2026-04-05
- Architecture Decision Records, https://github.com/joelparkerhenderson/architecture-decision-record, consulted 2026-04-05

### Technology Stack
- VS Code Copilot Chat agents documentation, https://code.visualstudio.com/docs/copilot, consulted 2026-04-05

### Security
- OWASP Secure Coding Practices, https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/, consulted 2026-04-05

### Standards & Specifications
- API-First Approach, https://swagger.io/resources/articles/adopting-an-api-first-approach/, consulted 2026-04-05
- Microsoft REST API Design, https://learn.microsoft.com/en-us/azure/architecture/microservices/design/api-design, consulted 2026-04-05

---

## 18. Version History

| Version | Date | Author | Summary of Changes |
|---------|------|--------|--------------------|
| 1.0 | 2026-04-05 | Spec Architect | Initial specification |
| 1.0.1 | 2026-04-05 | Spec Architect | Self-review corrections: ensured all FRs use SHALL, verified traceability matrix completeness, confirmed no ambiguous language |
