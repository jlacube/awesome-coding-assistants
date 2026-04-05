# Planner V2 - Skill-Based Contract-Generating Decomposition -- Specification

> **Source brief**: `.sdd/ideas/002-sdd-pipeline-v2-universal-skill-architecture.md`
> **Feature branch**: `003-planner-v2`
> **Status**: Validated
> **Version**: 1.1

---

## 1. Overview

Replace the monolithic Planner agent (single-pass work package decomposition) with a skill-based Planner coordinator that dispatches 8 sequential skills, each responsible for one planning/contract-generation dimension. The Planner V2 operates in two phases: Phase 1 decomposes the spec into work packages and tasks; Phase 2 generates language-specific contract files (interfaces, data schemas, API contracts, state machines, error catalogs, config schemas) per work package in 800-line blocks. Contracts are stored in `.sdd/plans/contracts/` and serve as the single source of truth that connects the spec to the Coder. When the Planner discovers spec gaps, it auto-loops to the Spec Architect (max 3 iterations) rather than flagging to the human.

---

## 2. Goals & Success Criteria

- **SC-001**: Every work package task references concrete contract artifacts (interface files, schema files) rather than prose-only spec sections. Verified by: every task in a WP file has at least one contract file reference.
- **SC-002**: Contract files contain valid syntax in the target language and can be imported/used by the Coder verbatim. Verified by: syntax validation of all generated contract files.
- **SC-003**: Spec gaps discovered during decomposition are resolved autonomously via auto-loop to Spec Architect, with human escalation only after 3 failed iterations. Verified by: gap resolution log in plan README.
- **SC-004**: Each planning skill runs in a fresh context window, enabling deeper analysis per dimension. Verified by: each subagent loads only its SKILL.md plus the plan accumulator -- no other skill instructions compete for context.
- **SC-005**: Cross-WP consistency (data contracts, API contracts, dependencies, config) is validated before presenting to the user. Verified by: no contract mismatches between WPs.
- **SC-006**: Adding a new planning dimension requires creating exactly one skill file in `.github/skills/plan-*/SKILL.md`. Verified by: no coordinator edit needed to add a planning dimension.

---

## 3. Users & Roles

- **Human Developer (approver)**: Reviews and approves the plan at the human gate. Needs: WPs with clear task boundaries, concrete contract references, dependency graphs, and acceptance criteria traceable to the spec.

- **Spec Architect Agent (upstream producer + gap resolution target)**: Produces the spec and companion artifacts that the Planner decomposes. Receives auto-loop requests when the Planner discovers gaps.

- **Coder Agent (downstream consumer)**: Picks up WPs and implements against contract files. Needs: implementation-complete tasks with exact function signatures, data schemas, API contracts, error codes, and validation rules in the target language.

- **Review Coordinator (downstream consumer)**: Uses WP task acceptance criteria and contract files to validate implementation. Needs: traceability from task -> FR -> contract -> test.

- **Orchestrator Agent (invoker)**: Triggers the Planner when a validated spec exists without a plan. Reads plan status to determine pipeline state.

---

## 4. Functional Requirements

### 4.1 Planner Coordinator

#### 4.1.1 Spec Selection and Validation

- **FR-001**: The coordinator SHALL list all files in `.sdd/specs/` and present them to the user for selection. If only one spec exists, the coordinator SHALL confirm it before proceeding.
  - Error: If `.sdd/specs/` is empty, the coordinator SHALL inform the user and halt.

- **FR-002**: The coordinator SHALL read the selected spec in full, including any companion artifact files in `.sdd/specs/artifacts/<NNN>-<idea-name>/`.

- **FR-003**: The coordinator SHALL verify the spec's `Status` field is "Validated" or "Final". If the status is "Draft", the coordinator SHALL refuse to proceed and recommend handing off to the Spec Architect.
  - Error: Draft spec - halt, recommend Spec Architect handoff.

#### 4.1.2 Spec Completeness Pre-Check

- **FR-004**: Before decomposition, the coordinator SHALL run a completeness pre-check against the spec:
  1. Traceability matrix (Section 16) has no empty cells
  2. Every FR has defined error behavior
  3. Every entity field has type, constraints, and validation rules
  4. Every API endpoint has all applicable error codes
  5. Every external integration has timeout/retry/fallback strategy
  6. Every entity with a status field has explicit state transitions
  7. Cross-cutting concerns (auth, logging, pagination, rate limiting) are addressed
  - If ANY check fails, the coordinator SHALL create a gap report and auto-loop to the Spec Architect (FR-006).

- **FR-005**: The coordinator SHALL read spec companion artifacts (`.sdd/specs/artifacts/<NNN>-<idea-name>/`) and verify they are consistent with the prose spec. If artifacts are missing or inconsistent, the coordinator SHALL include this in the gap report.

#### 4.1.3 Auto-Loop to Spec Architect

- **FR-006**: When the coordinator discovers spec gaps, it SHALL auto-loop to the Spec Architect via `runSubagent` with:
  1. The gap report listing every incomplete or inconsistent item
  2. The spec file path
  3. The companion artifacts directory path
  4. A request to resolve the gaps and update the spec
  - The coordinator SHALL retry up to 3 times. After 3 failed iterations (gaps still present), the coordinator SHALL escalate to the human with the gap report.
  - Error: Spec Architect subagent failure - escalate to human with full context.
  - Postcondition: After a successful auto-loop, the coordinator re-reads the spec and re-runs the completeness pre-check.

#### 4.1.4 Research Phase

- **FR-007**: The coordinator SHALL invoke a workspace research subagent to discover:
  1. Existing code, project structure, build system, and test frameworks
  2. Existing patterns, conventions, and infrastructure
  3. Any existing `.sdd/plans/` or `.sdd/docs/` that inform sequencing
  - The research subagent SHALL NOT draft any plan content.

- **FR-008**: The coordinator SHALL conduct web research:
  1. Official documentation for libraries/frameworks in the tech stack
  2. Known gotchas, migration issues, or common mistakes with chosen technologies
  3. Testing framework guides for the target stack
  4. Starter templates or boilerplate repos matching the tech stack

#### 4.1.5 Patterns Consumption

- **FR-009**: Before dispatching any skill, the coordinator SHALL read `.sdd/reviews/plan-patterns.md` (if it exists) and include active pattern summaries in the prompt for each skill. Skills SHALL avoid producing plan content that would trigger known patterns.

#### 4.1.6 Dynamic Skill Discovery

- **FR-010**: The coordinator SHALL discover available planning skills by scanning for directories matching the glob pattern `.github/skills/plan-*/SKILL.md` at the start of each plan generation.
  - Postcondition: A sorted list of discovered skill names is produced.
  - Error: If zero skills are discovered, the coordinator SHALL halt and report that no planning skills are installed.

- **FR-011**: The coordinator SHALL dispatch discovered skills in a deterministic order. The canonical order is:

  **Phase 1 -- Decomposition**:
  1. `plan-decomposition` (WP identification, task breakdown, sequencing, dependencies)
  2. `plan-acceptance` (acceptance criteria extraction, spec traceability per task)

  **Phase 2 -- Contract Generation**:
  3. `plan-interface-contracts` (public function/method signatures per WP)
  4. `plan-data-schemas` (entity type definitions, validation rules per WP)
  5. `plan-api-contracts` (request/response types, endpoint definitions per WP)
  6. `plan-state-machines` (state enums, transition validators per WP)
  7. `plan-error-catalogs` (error code constants, messages per WP)
  8. `plan-cross-wp-validation` (cross-WP consistency check, config schemas)

  Skills not present are skipped. Skills present but not in this list are dispatched after all known skills, in alphabetical order.

#### 4.1.7 Two-Phase Execution

- **FR-012**: The coordinator SHALL execute Phase 1 (skills 1-2) first, producing the plan accumulator (WP files and README). Phase 2 (skills 3-8) then reads the plan accumulator and generates contract files.
  - Phase 1 output: `.sdd/plans/README.md` + `.sdd/plans/WP<NN>-<slug>.md` files.
  - Phase 2 output: `.sdd/plans/contracts/<WP-slug>/` directories with language-specific contract files.

- **FR-013**: Phase 2 skills SHALL generate contracts in 800-line blocks per WP to manage context window limits. If a WP requires more than 800 lines of contracts, the skill SHALL split across multiple invocations, reading its prior output between blocks.

#### 4.1.8 Skill Dispatch

- **FR-014**: The coordinator SHALL dispatch each skill as a subagent invocation using `runSubagent`. Each invocation SHALL include:
  1. The skill file path
  2. The plan accumulator paths (README + WP files)
  3. The spec file path and companion artifacts directory
  4. Research findings summary
  5. The target language for contracts
  6. Active plan-domain patterns to avoid
  7. Phase indicator (1 or 2)
  - Error: If a Phase 1 skill fails, halt immediately (decomposition is sequential and dependent). If a Phase 2 contract skill fails, log the error and continue to the next skill (contracts are independent per dimension).

- **FR-015**: Phase 1 skills SHALL execute sequentially, one at a time, blocking. Phase 2 contract skills SHALL also execute sequentially but MAY skip failed skills.

- **FR-016**: Each skill SHALL read the current state of the plan accumulator before writing. This enables context forwarding between skills.

#### 4.1.9 Plan Accumulator Initialization

- **FR-017**: Before dispatching Phase 1 skills, the coordinator SHALL create:
  1. The plan directory `.sdd/plans/` (if it does not exist)
  2. The contracts directory `.sdd/plans/contracts/` (if it does not exist)
  3. A skeleton README at `.sdd/plans/README.md` with: spec reference, target language, plan status ("In Progress")

#### 4.1.10 Post-Completion Validation

- **FR-018**: After all skills complete, the coordinator SHALL run a cross-WP consistency audit:
  1. Data contract consistency: entity field names, types, validation rules match across WPs
  2. API/Interface contract consistency: signatures match between producer and consumer WPs
  3. Dependency integrity: no circular dependencies, valid dependency references
  4. Configuration consistency: env vars, config keys, secrets use identical names and types
  5. Test consistency: coverage requirements stated consistently (80% code, 90% branch)
  6. Spec traceability: every FR assigned to exactly one task, no orphan FRs, no duplicate assignments
  7. Contract file consistency: contract files match their corresponding WP task specifications
  - If any inconsistency is found, the coordinator SHALL fix it and document corrections in README under "Consistency Notes".

- **FR-019**: The coordinator SHALL verify every WP file is implementation-complete:
  1. 5-12 tasks per WP
  2. At least 3 acceptance criteria per task
  3. Implementation guidance with official doc links per task
  4. Contract file references per task
  5. No ambiguous language ("should", "appropriate", "reasonable")
  - If any WP fails validation, the coordinator SHALL fix it inline.

#### 4.1.11 Presentation and Approval

- **FR-020**: After validation, the coordinator SHALL present the plan to the user. The files are for persistence; the coordinator SHALL also show plan content in the chat.

- **FR-021**: On user feedback:
  - Changes requested: revise WPs and re-validate, then re-present
  - Questions asked: clarify or ask follow-ups
  - Approval: acknowledge, recommend Coder agent for WP01

#### 4.1.12 Commit Policy

- **FR-022**: Each WP file SHALL be committed individually after completion:
  ```
  git add .sdd/plans/WP<NN>-<slug>.md
  git commit -m "docs(plan): add WP<NN> <title>"
  ```
  README SHALL be committed as a standalone change.
  Contract files SHALL be committed per-WP:
  ```
  git add .sdd/plans/contracts/<WP-slug>/
  git commit -m "docs(plan): add contracts for WP<NN>"
  ```
  Files SHALL be listed explicitly in `git add`.

#### Implementation Contract -- Planner Coordinator

**Inputs**:
- Spec file path (string, resolved from `.sdd/specs/`): the validated specification to decompose.
- Spec companion artifacts directory (string): language-specific artifacts from Spec Architect.
- User answers (interactive): responses to alignment questions.

**Outputs**:
- Plan README: `.sdd/plans/README.md` with WP index, dependency graph, MVP scope, consistency notes.
- WP files: `.sdd/plans/WP<NN>-<slug>.md` with tasks, acceptance criteria, implementation guidance.
- Contract files: `.sdd/plans/contracts/<WP-slug>/` with language-specific contracts per WP.
- Git commits for all files.

**Error behaviors**:
- No specs in `.sdd/specs/`: halt, inform user.
- Spec status not Validated: halt, recommend Spec Architect handoff.
- Spec completeness check fails: auto-loop to Spec Architect (max 3), then escalate to human.
- Phase 1 skill failure: halt immediately.
- Phase 2 skill failure: log and continue to next skill.
- Cross-WP validation failures: fix inline and document.

---

### 4.2 Planning Skills (Common Contract)

#### 4.2.1 Skill Input Contract

- **FR-023**: Every planning skill SHALL accept the following inputs in its subagent prompt:
  1. `skill_path`: Path to its SKILL.md file
  2. `plan_dir`: Path to `.sdd/plans/` directory
  3. `contracts_dir`: Path to `.sdd/plans/contracts/` directory
  4. `spec_path`: Path to the source spec file
  5. `spec_artifacts_dir`: Path to spec companion artifacts
  6. `research_summary`: Key findings from the research phase
  7. `target_language`: Programming language for contract generation
  8. `patterns`: Active plan-domain patterns to avoid
  9. `phase`: Phase indicator (1 or 2)

- **FR-024**: Every planning skill SHALL, upon invocation:
  1. Read its own SKILL.md
  2. Read the current plan state (README, WP files, existing contracts)
  3. Read the source spec and companion artifacts
  4. Write its assigned artifacts to the plan directory or contracts directory

#### 4.2.2 Skill Output Contract

- **FR-025**: Phase 1 skills SHALL write WP files and README to `.sdd/plans/`.
- **FR-026**: Phase 2 skills SHALL write contract files to `.sdd/plans/contracts/<WP-slug>/`.
- **FR-027**: Skills SHALL NOT modify files written by earlier skills unless explicitly needed for consistency fixes, marked with `[CONSISTENCY FIX: <description>]`.

---

### 4.3 Decomposition Skill (plan-decomposition) -- Phase 1

- **FR-028**: The plan-decomposition skill SHALL:
  1. Analyze the spec's functional requirements, user stories, and architecture
  2. Identify logical work packages following the sequencing logic: Foundation -> Core domain -> Integrations -> User-facing -> Quality -> Delivery
  3. Decompose each WP into 5-12 atomic tasks
  4. Define inter-task and inter-WP dependencies by ID
  5. Assign priorities: P0 (foundation), P1 (MVP user story), P2+ (incremental)
  6. Write WP files with the standard template (metadata table, objective, spec references, task list)
  7. Write skeleton README with WP index and dependency graph

- **FR-029**: Each task SHALL include:
  1. Unique T<NN>-XX identifier
  2. Description (what must be done, precisely)
  3. Spec refs (FR-XXX, Section N.X)
  4. Parallel flag (can it run concurrently within the WP?)
  5. Placeholder for acceptance criteria (filled by plan-acceptance skill)
  6. Placeholder for implementation guidance (filled by plan-acceptance skill)
  7. Dependencies on other tasks

- **FR-030**: The skill SHALL target 5-12 tasks per WP. Fewer than 5 suggests the WP is too granular; more than 12 suggests it should be split.

- **FR-031**: Every WP file SHALL include a metadata table header:
  ```markdown
  | Field | Value |
  |-------|-------|
  | Spec | `.sdd/specs/<NNN>-<name>.spec.md` |
  | Priority | P0 / P1 / P2 |
  | Lane | planned |
  | Depends on | WP<NN> or none |
  | Goal | One-sentence user-observable outcome |
  | Status | Not Started |
  | Independent Test | How to verify in isolation |
  ```

- **FR-032**: The first task of the foundation WP (WP01 or equivalent) SHALL always be virtual environment setup for languages with package isolation (Python: venv/poetry/conda, Node: local node_modules).

#### Implementation Contract -- plan-decomposition

**Inputs**: Spec + artifacts, research summary, patterns.
**Outputs**: WP files (`.sdd/plans/WP<NN>-<slug>.md`) with task skeletons, README skeleton.
**Artifacts**: None (prose plan files).

---

### 4.4 Acceptance Criteria Skill (plan-acceptance) -- Phase 1

- **FR-033**: The plan-acceptance skill SHALL populate every task's acceptance criteria and implementation guidance:
  1. Copy exact SHALL statements from the spec's FRs as acceptance criteria (at least 3 per task)
  2. Copy acceptance scenarios from the spec's user stories (Given/When/Then)
  3. Add implementation guidance with official doc links, recommended patterns, known pitfalls, error codes, and validation rules
  4. Specify test requirements per task (unit / integration / BDD / E2E / none)

- **FR-034**: The skill SHALL verify traceability: every FR in the spec's traceability matrix (Section 16) is assigned to exactly one task. If an FR is unassigned, the skill SHALL assign it. If an FR is assigned to multiple tasks, the skill SHALL resolve the duplication.

- **FR-035**: The skill SHALL include BDD/TDD requirements in every task with test requirements: tests derive from spec acceptance scenarios, not from implementation.

- **FR-036**: The skill SHALL update the README with:
  1. Complete WP index with status, priority, dependencies
  2. MVP scope (which WPs constitute the minimum releasable increment)
  3. Dependency graph

#### Implementation Contract -- plan-acceptance

**Inputs**: Plan accumulator (WP files with task skeletons), spec + artifacts.
**Outputs**: Updated WP files with full acceptance criteria and implementation guidance, updated README.
**Artifacts**: None.

---

### 4.5 Interface Contracts Skill (plan-interface-contracts) -- Phase 2

- **FR-037**: The plan-interface-contracts skill SHALL generate interface contract files for each WP:
  1. Public function/method signatures with typed parameters and return types
  2. Class/module interface definitions (abstract classes, protocols, traits)
  3. Module export definitions
  - File: `.sdd/plans/contracts/<WP-slug>/interfaces.<ext>`

- **FR-038**: Interface signatures SHALL match the spec's companion artifact `interfaces.<ext>` from `.sdd/specs/artifacts/`. The skill SHALL read the spec artifact and extend it with WP-specific implementation details (e.g., internal helper signatures that the spec does not define but the WP needs).

- **FR-039**: Each contract file SHALL include a manifest header:
  ```
  // Generated by: plan-interface-contracts skill
  // Source spec: .sdd/specs/<NNN>-<name>.spec.md
  // Work package: WP<NN>-<slug>
  // Target language: <language>
  // DO NOT EDIT MANUALLY -- regenerated on plan revision
  ```

#### Implementation Contract -- plan-interface-contracts

**Inputs**: Plan accumulator, spec + artifacts, target language.
**Outputs**: `.sdd/plans/contracts/<WP-slug>/interfaces.<ext>` per WP.

---

### 4.6 Data Schemas Skill (plan-data-schemas) -- Phase 2

- **FR-040**: The plan-data-schemas skill SHALL generate data schema contract files for each WP:
  1. Entity type definitions (classes, interfaces, structs, dataclasses) with all fields typed
  2. Field validation rules as code (decorators, validators, constraints)
  3. Relationship definitions
  - File: `.sdd/plans/contracts/<WP-slug>/data-schemas.<ext>`

- **FR-041**: Data schemas SHALL match the spec's companion artifact `data-models.<ext>` from `.sdd/specs/artifacts/`. The skill SHALL scope each WP's schema to only the entities that WP creates or modifies.

- **FR-042**: If an entity is shared across multiple WPs, the first WP to define it SHALL include the full definition. Subsequent WPs SHALL import or reference the definition from the first WP's contracts.

#### Implementation Contract -- plan-data-schemas

**Inputs**: Plan accumulator, spec + artifacts, target language.
**Outputs**: `.sdd/plans/contracts/<WP-slug>/data-schemas.<ext>` per WP that touches data models.

---

### 4.7 API Contracts Skill (plan-api-contracts) -- Phase 2

- **FR-043**: The plan-api-contracts skill SHALL generate API contract files for each WP:
  1. Request type definitions with all fields typed and validated
  2. Response type definitions with all fields typed
  3. Endpoint path constants
  4. Auth requirement declarations per endpoint
  - File: `.sdd/plans/contracts/<WP-slug>/api-contracts.<ext>`

- **FR-044**: API contracts SHALL match the spec's companion artifact `api-contracts.<ext>` from `.sdd/specs/artifacts/`. Scoped per WP.

- **FR-045**: Each endpoint contract SHALL include all applicable error response types (400, 401, 403, 404, 409, 422, 500) with typed error response schemas.

#### Implementation Contract -- plan-api-contracts

**Inputs**: Plan accumulator, spec + artifacts, target language.
**Outputs**: `.sdd/plans/contracts/<WP-slug>/api-contracts.<ext>` per WP with API endpoints.

---

### 4.8 State Machines Skill (plan-state-machines) -- Phase 2

- **FR-046**: The plan-state-machines skill SHALL generate state machine contract files for each WP that creates or modifies stateful entities:
  1. State enum definitions with all valid values
  2. Transition validation functions (from-state, to-state, guard)
  3. Side effect declarations per transition
  - File: `.sdd/plans/contracts/<WP-slug>/state-machines.<ext>`

- **FR-047**: State machines SHALL match the spec's companion artifact `state-machines.<ext>` from `.sdd/specs/artifacts/`. Scoped per WP.

#### Implementation Contract -- plan-state-machines

**Inputs**: Plan accumulator, spec + artifacts, target language.
**Outputs**: `.sdd/plans/contracts/<WP-slug>/state-machines.<ext>` per WP with stateful entities.

---

### 4.9 Error Catalogs Skill (plan-error-catalogs) -- Phase 2

- **FR-048**: The plan-error-catalogs skill SHALL generate error catalog contract files for each WP:
  1. Error code constants/enums
  2. HTTP status code mappings (if applicable)
  3. User-facing error message templates
  4. Internal log message templates
  - File: `.sdd/plans/contracts/<WP-slug>/error-catalog.<ext>`

- **FR-049**: Error catalogs SHALL match the spec's companion artifact `error-catalog.<ext>` from `.sdd/specs/artifacts/`. Scoped per WP.

- **FR-050**: If an error code is shared across multiple WPs, the first WP to define it SHALL include the full definition. Subsequent WPs SHALL import or reference from the first WP.

#### Implementation Contract -- plan-error-catalogs

**Inputs**: Plan accumulator, spec + artifacts, target language.
**Outputs**: `.sdd/plans/contracts/<WP-slug>/error-catalog.<ext>` per WP.

---

### 4.10 Cross-WP Validation Skill (plan-cross-wp-validation) -- Phase 2

- **FR-051**: The plan-cross-wp-validation skill SHALL perform a comprehensive consistency audit across all WPs and contracts:
  1. **Data contract consistency**: entity fields match across WPs
  2. **API/Interface contract consistency**: signatures match between producer and consumer WPs
  3. **Dependency integrity**: no circular dependencies, valid references
  4. **Configuration consistency**: env vars, config keys use identical names/types across WPs
  5. **Test consistency**: coverage requirements (80% code, 90% branch) stated consistently
  6. **Spec traceability**: every FR assigned to exactly one task, no orphans
  7. **Contract-to-task alignment**: every contract file is referenced by at least one task

- **FR-052**: The skill SHALL also generate a configuration schema contract file:
  - All environment variables referenced across all WPs: name, type, default, validation, description
  - File: `.sdd/plans/contracts/shared/config-schema.<ext>`

- **FR-053**: If inconsistencies are found, the skill SHALL fix them in the relevant WP and contract files, and document all corrections in the README under "Consistency Notes".

- **FR-054**: The skill SHALL verify that the generated contract files collectively cover 100% of the spec's companion artifacts. No spec artifact entity, endpoint, error code, or state machine may be missing from the plan's contracts.

#### Implementation Contract -- plan-cross-wp-validation

**Inputs**: All plan files, all contract files, spec + artifacts.
**Outputs**: Updated files (consistency fixes), `config-schema.<ext>` in `contracts/shared/`, README updates.

---

## 5. User Stories

### US-01 -- Decompose a Spec into Implementation-Ready WPs (Priority: P1) MVP

**As a** Human Developer, **I want** the Planner to decompose a validated spec into sequenced work packages with concrete task definitions, **so that** the Coder can implement each task without asking clarifying questions.

**Why P1**: This is the core planning function -- without it, no implementation can begin.

**Independent Test**: Provide a validated spec. Invoke the Planner. Verify: WP files exist with 5-12 tasks each, every task has 3+ acceptance criteria, implementation guidance with doc links, no ambiguous language.

**Acceptance Scenarios**:
1. **Given** a validated spec with 20 FRs, **When** the Planner completes Phase 1, **Then** all 20 FRs are assigned to exactly one task across the WPs, with no orphans.
2. **Given** a spec with a foundation requirement (project scaffolding), **When** the Planner decomposes, **Then** WP01 is a P0 foundation WP with virtual environment setup as T01-01.
3. **Given** a spec with independent feature groups, **When** the Planner decomposes, **Then** independent WPs are marked "Parallelisable: Yes".

---

### US-02 -- Generate Language-Specific Contract Files (Priority: P1) MVP

**As the** Coder agent, **I want** contract files in the target language for each WP, **so that** I can implement against precise interfaces, schemas, and error definitions without interpreting prose.

**Why P1**: Contract-first implementation is the primary mechanism to eliminate drift.

**Independent Test**: After plan generation, verify: `.sdd/plans/contracts/<WP-slug>/` directories exist for each WP, contract files contain valid target-language syntax, field names match the spec exactly.

**Acceptance Scenarios**:
1. **Given** a spec with 5 entities and 10 endpoints in TypeScript, **When** Phase 2 completes, **Then** each WP has `data-schemas.ts`, `api-contracts.ts`, and `error-catalog.ts` in its contracts directory.
2. **Given** a WP that does not touch the data model, **When** Phase 2 runs, **Then** no `data-schemas.<ext>` is generated for that WP.
3. **Given** a shared entity used by WP02 and WP04, **When** Phase 2 runs, **Then** WP02 defines the entity, and WP04 imports/references it from WP02's contracts.

---

### US-03 -- Auto-Resolve Spec Gaps (Priority: P1) MVP

**As a** Human Developer, **I want** the Planner to resolve spec gaps autonomously by looping to the Spec Architect, **so that** I am not interrupted for issues the pipeline can fix itself.

**Why P1**: Reduces human intervention, making the pipeline more autonomous.

**Independent Test**: Provide a spec missing error behaviors for 3 FRs. Run the Planner. Verify: the Planner auto-loops to Spec Architect, gaps are resolved, and the plan proceeds without human intervention.

**Acceptance Scenarios**:
1. **Given** a spec missing error behavior for FR-005, **When** the Planner runs the completeness pre-check, **Then** it creates a gap report and invokes the Spec Architect subagent to resolve it.
2. **Given** the Spec Architect resolves the gap on the first attempt, **When** the Planner re-checks, **Then** it proceeds to decomposition without human escalation.
3. **Given** the Spec Architect fails to resolve a gap after 3 attempts, **When** the Planner re-checks, **Then** it escalates to the human with the gap report.

---

### US-04 -- Cross-WP Consistency Validation (Priority: P1) MVP

**As a** Human Developer, **I want** the Planner to validate consistency across WPs before presenting the plan, **so that** I review a coherent plan without cross-WP contradictions.

**Why P1**: Cross-WP inconsistencies are the #1 cause of integration failures.

**Independent Test**: Introduce a deliberate inconsistency (e.g., WP02 uses field "userId" but WP04 uses "user_id"). Verify: the cross-WP validation skill catches and fixes it.

**Acceptance Scenarios**:
1. **Given** WP02 defines entity User with field "email" and WP04 references "emailAddress", **When** cross-WP validation runs, **Then** the inconsistency is detected and resolved.
2. **Given** WP03 depends on WP05 and WP05 depends on WP03 (circular), **When** cross-WP validation runs, **Then** the circular dependency is detected and flagged.
3. **Given** all WPs are consistent, **When** cross-WP validation runs, **Then** no corrections are needed and a clean report is generated.

---

### US-05 -- Dynamic Planning Skill Discovery (Priority: P2)

**As a** system maintainer, **I want** to add a new planning dimension by creating a `.github/skills/plan-<name>/SKILL.md` file, **so that** the Planner dispatches it automatically.

**Why P2**: Extensibility for future planning dimensions without coordinator changes.

**Independent Test**: Add a `.github/skills/plan-compliance/SKILL.md` file. Run the Planner. Verify: the new skill is discovered and dispatched after all known skills.

**Acceptance Scenarios**:
1. **Given** 8 planning skills installed, **When** a 9th skill `plan-compliance` is added, **Then** 9 skills are dispatched.
2. **Given** a skill `plan-foo` is removed, **When** the Planner runs, **Then** remaining skills dispatch without error.

---

### Edge Cases

- What happens when a spec has zero entities (pure logic, no data model)? Phase 2 data and state machine skills produce no output for WPs without entities. The cross-WP validation skill confirms this is intentional.
- What happens when the 800-line contract block limit is hit? The skill splits across multiple writes, reading prior output between blocks to maintain consistency.
- What happens when the Spec Architect is unavailable for auto-loop? After timeout or failure, escalate to the human with the gap report.
- What happens when a WP has tasks that span more than 12? The decomposition skill splits it into two WPs with explicit dependency.

---

## 6. User Flows

### 6.1 Full Plan Generation Flow

1. User or Orchestrator invokes the Planner.
2. Coordinator lists specs in `.sdd/specs/` and asks user to select (or confirms single spec).
3. Coordinator reads spec + companion artifacts.
4. Coordinator verifies spec status is Validated.
5. Coordinator runs completeness pre-check (FR-004).
6. If gaps found: auto-loop to Spec Architect (FR-006), max 3 times.
7. If gaps persist after 3 loops: escalate to human, halt.
8. Coordinator invokes workspace research subagent.
9. Coordinator conducts web research.
10. Coordinator reads plan-patterns.md (if exists).
11. Coordinator creates plan directory, contracts directory, skeleton README.
12. Coordinator discovers planning skills via glob scan.
13. **Phase 1** -- Decomposition:
    a. Dispatch plan-decomposition (WP files + README skeleton).
    b. Dispatch plan-acceptance (acceptance criteria, implementation guidance, traceability).
    c. Commit each WP file individually.
14. **Phase 2** -- Contract Generation:
    a. For each WP, dispatch plan-interface-contracts.
    b. For each WP, dispatch plan-data-schemas.
    c. For each WP, dispatch plan-api-contracts.
    d. For each WP, dispatch plan-state-machines.
    e. For each WP, dispatch plan-error-catalogs.
    f. Dispatch plan-cross-wp-validation (consistency audit + config schema).
    g. Commit contracts per WP.
15. Coordinator runs post-completion validation (FR-018, FR-019).
16. Coordinator presents plan to user.
17. User reviews:
    a. Approval: acknowledge, recommend Coder for WP01.
    b. Changes: revise, re-validate, re-present.
    c. Questions: clarify.

### 6.2 Spec Gap Auto-Loop Flow

1. Completeness pre-check discovers gaps (e.g., FR-012 missing error behavior).
2. Coordinator creates gap report markdown.
3. Coordinator invokes Spec Architect subagent with gap report.
4. Spec Architect updates the spec file and companion artifacts.
5. Coordinator re-reads spec and re-runs completeness pre-check.
6. If gaps remain: increment attempt counter, repeat from step 3 (max 3).
7. If gaps persist after 3 attempts: present gap report to human, halt.

---

## 7. Data Model

### 7.1 Plan Accumulator Files

**README**: `.sdd/plans/README.md`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| Spec reference | string | valid path to `.sdd/specs/` | Source spec for this plan |
| Target language | string | programming language | Language for contract generation |
| Plan status | enum | `In Progress`, `Ready`, `Approved` | Current plan state |
| WP index | table | WP ID, title, priority, status, depends on | Summary of all WPs |
| Dependency graph | mermaid/prose | acyclic | Visual dependency structure |
| MVP scope | list | WP IDs | Minimum releasable increment |
| Consistency notes | text | optional | Cross-WP fixes applied |

**WP Files**: `.sdd/plans/WP<NN>-<slug>.md`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| lane | enum | `planned`, `doing`, `for_review`, `done`, `to_do` | YAML frontmatter lifecycle state |
| Spec | string | valid spec path | Source spec |
| Priority | enum | P0, P1, P2, P3 | P0=foundation, P1=MVP, P2+=incremental |
| Lane | enum | planned, doing, for_review, done, to_do | Current lifecycle state |
| Depends on | string | WP IDs or "none" | Dependencies |
| Goal | string | 1 sentence | User-observable outcome |
| Status | string | Not Started, In Progress, Done | Execution status |
| Independent Test | string | action + observable result | Isolation verification |
| Parallelisable | boolean | Yes/No | Can run concurrently with other WPs |
| Objective | text | 1 paragraph | What this WP delivers |
| Spec References | list | FR-XXX, Section N.X | Spec sections covered |
| Tasks | list(Task) | 5-12 per WP | Atomic implementable tasks |

**Task within WP**:

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| ID | string | T<NN>-XX format | Unique task identifier |
| Description | text | precise, no ambiguity | What must be done |
| Spec refs | list(string) | FR-XXX, Section N.X | Source spec requirements |
| Parallel | boolean | Yes/No | Concurrent within WP |
| Acceptance criteria | list(string) | min 3 items, from spec SHALL statements | Pass/fail conditions |
| Test requirements | enum(s) | unit, integration, BDD, E2E, none | Required test types |
| Depends on | string | T<NN>-XX or "none" | Task dependencies |
| Implementation Guidance | object | doc links, patterns, pitfalls, error codes | Coder reference |
| Contract refs | list(string) | file paths in contracts/ | Contract files for this task |

### 7.2 Contract Files

**Directory**: `.sdd/plans/contracts/<WP-slug>/`

| File | Type | Content |
|------|------|---------|
| `interfaces.<ext>` | target language | Public function/method signatures |
| `data-schemas.<ext>` | target language | Entity type definitions with validation |
| `api-contracts.<ext>` | target language | Request/response types, endpoint paths |
| `state-machines.<ext>` | target language | State enums, transition validators |
| `error-catalog.<ext>` | target language | Error code constants, messages |

**Shared directory**: `.sdd/plans/contracts/shared/`

| File | Type | Content |
|------|------|---------|
| `config-schema.<ext>` | target language | All env vars with types, defaults, validation |

### 7.3 Gap Report

Generated by the coordinator when spec completeness pre-check fails. Passed to Spec Architect via auto-loop.

| Field | Type | Description |
|-------|------|-------------|
| Spec path | string | Path to the spec being checked |
| Gap ID | string | GAP-XXX sequential identifier |
| Category | enum | traceability, error-behavior, data-validation, api-errors, integration-failure, state-machine, cross-cutting |
| FR reference | string | The FR that is incomplete |
| Description | text | What is missing |
| Impact | text | How this blocks planning |

---

## 8. API / Interface Design

### 8.1 Coordinator Invocation Interface

**Invocation methods**:
1. Direct: user selects "Planner" agent mode
2. Handoff: Orchestrator delegates when a validated spec exists without a plan
3. Handoff: Spec Architect hands off after spec is validated

**Response**: Markdown-formatted plan presented in VS Code chat, plus file modifications (WP files, README, contract files).

### 8.2 Skill Subagent Prompt Template (Phase 1)

```
Execute Phase 1 planning: <skill_name>

1. Read the skill instructions at: <skill_path>
2. Read the spec at: <spec_path>
3. Read spec companion artifacts at: <spec_artifacts_dir>
4. Read existing plan state at: <plan_dir>
5. Research context: <research_summary>
6. Active patterns to avoid: <patterns>
7. Target language: <target_language>

Write plan files to <plan_dir>.

Rules:
- Read existing plan files to maintain consistency with prior skills
- Every task must trace to a spec FR
- At least 3 acceptance criteria per task
- Include implementation guidance with official doc links
- 5-12 tasks per WP
- Use [NEEDS CLARIFICATION] for unresolved items
```

### 8.3 Skill Subagent Prompt Template (Phase 2)

```
Execute Phase 2 contract generation: <skill_name>

1. Read the skill instructions at: <skill_path>
2. Read the spec at: <spec_path> and artifacts at: <spec_artifacts_dir>
3. Read the plan at: <plan_dir> (README + WP files)
4. Target language: <target_language>
5. Contracts directory: <contracts_dir>

For each WP that this skill applies to, generate contract files in <contracts_dir>/<WP-slug>/.

Rules:
- Contract field names and types MUST match spec companion artifacts exactly
- Include manifest header in every contract file
- Stay within 800 lines per contract file; split if needed
- Scope contracts to the entities/endpoints that each WP creates or modifies
- Shared entities: first WP defines, subsequent WPs import/reference
```

### 8.4 Auto-Loop Prompt Template (to Spec Architect)

```
Spec gaps discovered during planning decomposition.

Spec: <spec_path>
Companion artifacts: <spec_artifacts_dir>

Gap Report:
<gap_report_markdown>

Please resolve these gaps by updating the spec and companion artifacts:
- Add missing error behaviors
- Define missing validation rules
- Complete the traceability matrix
- Add missing state transitions

This is auto-loop attempt <N> of 3.
```

### 8.5 Handoff Prompt Templates

**Start Implementation** (to Coder):
```
Plan approved. Work packages at: <plan_dir>
Contracts at: <contracts_dir>
Start with WP01.
```

**Clarify Specification** (to Spec Architect):
```
Spec gaps discovered during decomposition need resolution.
Gap report: <gap_report>
```

---

## 9. Architecture

### 9.1 System Design

```
User/Orchestrator
       |
       v
Planner Coordinator
       |
       |--> Spec selection + status validation
       |--> Spec completeness pre-check
       |     |
       |     |--> (gaps?) Auto-loop to Spec Architect (max 3)
       |
       |--> Research (subagent + web)
       |--> Read plan-patterns.md
       |--> Create plan + contracts directories
       |--> Discover skills (scan .github/skills/plan-*/)
       |
       |--- Phase 1: Decomposition ---
       |--> runSubagent(plan-decomposition)     --> WP files + README skeleton
       |--> runSubagent(plan-acceptance)         --> acceptance criteria + guidance + traceability
       |
       |--- Phase 2: Contract Generation ---
       |--> runSubagent(plan-interface-contracts)  --> interfaces per WP
       |--> runSubagent(plan-data-schemas)          --> data schemas per WP
       |--> runSubagent(plan-api-contracts)         --> API contracts per WP
       |--> runSubagent(plan-state-machines)        --> state machines per WP
       |--> runSubagent(plan-error-catalogs)        --> error catalogs per WP
       |--> runSubagent(plan-cross-wp-validation)   --> consistency + config schema
       |
       |--> Post-completion validation
       |--> Present to user
       |--> Commit on approval
```

### 9.2 Technology Stack

| Layer | Technology | Version | Rationale |
|-------|-----------|---------|-----------|
| Agent framework | VS Code Copilot Chat agents | Current | Existing SDD infrastructure |
| Skill framework | VS Code Copilot Chat skills | Current | Proven in Reviewer V2 and Spec Architect V2 |
| Data format | Markdown + YAML frontmatter | N/A | WP files with frontmatter for lifecycle tracking |
| Contract format | Target language source files | Varies | Eliminates interpretation; Coder copies verbatim |
| Version control | Git | Current | Explicit git add per file |

### 9.3 Directory & Module Structure

```
.github/
  agents/
    planner.agent.md                  # MODIFIED: refactored to coordinator pattern
  skills/
    plan-decomposition/SKILL.md       # NEW: WP identification, task breakdown
    plan-acceptance/SKILL.md          # NEW: acceptance criteria, traceability
    plan-interface-contracts/SKILL.md # NEW: function/method signatures
    plan-data-schemas/SKILL.md        # NEW: entity type definitions
    plan-api-contracts/SKILL.md       # NEW: request/response types
    plan-state-machines/SKILL.md      # NEW: state enums, transitions
    plan-error-catalogs/SKILL.md      # NEW: error code constants
    plan-cross-wp-validation/SKILL.md # NEW: consistency audit, config schema

.sdd/
  plans/
    README.md                         # Plan index, dependency graph, MVP scope
    WP<NN>-<slug>.md                  # Work package files
    contracts/
      <WP-slug>/
        interfaces.<ext>
        data-schemas.<ext>
        api-contracts.<ext>
        state-machines.<ext>
        error-catalog.<ext>
      shared/
        config-schema.<ext>
  reviews/
    plan-patterns.md                  # NEW: domain-specific patterns for planning
```

### 9.4 Key Design Decisions

**Decision 1: Two-phase execution (decompose first, contracts second)**
- **Rationale**: Contracts depend on task boundaries determined during decomposition. Generating contracts without knowing WP scope would produce mis-scoped output.
- **Alternatives considered**: Single-pass (contracts inline with tasks), contracts-first then decompose.
- **Consequences**: Phase 2 skills can read the complete plan before generating contracts, ensuring proper scoping.

**Decision 2: Contracts per WP (not per entity/endpoint)**
- **Rationale**: Coder processes one WP at a time. Having contracts scoped per WP means the Coder loads only what it needs.
- **Alternatives considered**: Single global contracts directory, contracts per entity, contracts per spec section.
- **Consequences**: Shared entities need cross-WP deduplication (first definer owns, others reference).

**Decision 3: Auto-loop to Spec Architect (max 3 iterations)**
- **Rationale**: Most spec gaps are minor (missing error behavior, incomplete validation rules). The Spec Architect can fix them faster than a human round-trip. The max-3 guard prevents infinite loops.
- **Alternatives considered**: Always escalate to human, Planner fixes gaps itself.
- **Consequences**: Spec Architect must support being invoked as a subagent for targeted fixes.

**Decision 4: 800-line block limit for contracts**
- **Rationale**: Large contract files may exceed context window limits. Splitting into blocks with read-between-writes maintains quality.
- **Alternatives considered**: No limit (risk quality degradation), 500-line limit (too granular), per-entity splitting.
- **Consequences**: Contract skills must handle multi-block generation with consistency between blocks.

**Decision 5: Phase 2 failure tolerance (unlike Phase 1)**
- **Rationale**: Phase 1 skills are dependent (acceptance criteria need tasks first). Phase 2 skills are independent per contract type -- a failed error catalog does not prevent interface generation.
- **Alternatives considered**: Halt on any failure, retry failed skills.
- **Consequences**: Some WPs may have incomplete contract coverage if a Phase 2 skill fails. The cross-WP validation skill catches this.

### 9.5 External Integrations

**Spec Architect (auto-loop)**:
- Purpose: Resolve spec gaps discovered during completeness pre-check.
- Authentication: None (local subagent).
- Key operations: Gap report submission, spec file update.
- Failure handling: Max 3 attempts, then escalate to human. Timeout after 10 minutes per attempt.

**Web research (via fetch_webpage)**:
- Purpose: Official docs, known pitfalls, testing frameworks.
- Authentication: None (public web).
- Failure handling: Note assumption and continue.

---

## 10. Non-Functional Requirements

### 10.1 Performance

- **NFR-001**: A full plan generation (8 skills, Phase 1 + Phase 2) SHALL complete within 60 minutes for a medium-complexity spec (5 WPs, 40 tasks, 15 entities, 20 endpoints).
- **NFR-002**: Phase 1 (decomposition + acceptance) SHALL complete within 20 minutes.
- **NFR-003**: Each Phase 2 contract skill SHALL complete within 5 minutes per WP.
- **NFR-004**: Spec gap auto-loop SHALL complete within 10 minutes per attempt (30 minutes max for 3 attempts).

### 10.2 Security

- **NFR-005**: Contract files SHALL NOT contain executable code beyond type definitions, interfaces, and validation schemas. No I/O, network access, or file system operations.
- **NFR-006**: No credentials, tokens, or API keys in plan or contract files. Configuration schemas SHALL define env var names and types but never values.

### 10.3 Scalability & Availability

- Local workspace only. No availability or scaling requirements.
- **NFR-007**: The system SHALL handle plans with up to 10 WPs and 120 tasks without quality degradation.
- **NFR-008**: The system SHALL handle up to 50 contract files across all WPs.

### 10.4 Accessibility

- Not applicable. Output is markdown and source code files.

### 10.5 Observability

- **Plan README**: Serves as the planning audit trail with WP index, dependency graph, and consistency notes.
- **WP frontmatter**: `lane` field tracks lifecycle status.
- **Commit history**: Each WP and contract committed separately for traceability.

---

## 11. Test Requirements

### 11.1 Unit Tests

Validation SHALL verify:
- Coordinator agent file has valid YAML frontmatter + valid markdown.
- Each skill file has valid YAML frontmatter + valid markdown.
- Contract files contain valid syntax in the target language.
- WP files have valid metadata tables.
- README has valid dependency graph structure.

### 11.2 BDD / Acceptance Tests

```gherkin
Feature: Planner V2 - Full Plan Generation

  Scenario: Decompose a validated spec into work packages with contracts
    Given a validated spec with 20 FRs and 5 entities
    And 8 planning skills are installed
    When the Planner is invoked and the user selects the spec
    Then Phase 1 produces WP files with 5-12 tasks each
    And Phase 2 produces contract files per WP
    And every FR is assigned to exactly one task
    And every contract file has a manifest header
    And cross-WP validation passes cleanly

  Scenario: Auto-loop resolves spec gaps on first attempt
    Given a spec missing error behavior for FR-005
    When the Planner runs completeness pre-check
    Then it creates a gap report and invokes Spec Architect
    And the Spec Architect resolves the gap
    And the Planner re-checks and proceeds to decomposition

  Scenario: Auto-loop escalates after 3 failed attempts
    Given a spec with a fundamental gap (missing entire entity definition)
    When the Spec Architect fails to resolve it 3 times
    Then the Planner escalates to the human with the full gap report
    And does not proceed to decomposition

  Scenario: Cross-WP validation catches inconsistency
    Given WP02 defines User.email and WP04 references User.emailAddress
    When plan-cross-wp-validation runs
    Then it detects the inconsistency
    And resolves it to a consistent field name
    And documents the fix in README Consistency Notes

  Scenario: Contract scoping per WP
    Given a spec with entity Order defined in WP02 and referenced in WP05
    When plan-data-schemas generates contracts
    Then WP02 contracts contain the full Order definition
    And WP05 contracts import/reference from WP02

  Scenario: Phase 2 skill failure does not halt pipeline
    Given plan-state-machines encounters an error
    When the coordinator detects the failure
    Then it logs the error and continues to plan-error-catalogs
    And plan-cross-wp-validation reports the missing state machine contracts

  Scenario: Dynamic planning skill discovery
    Given 8 planning skills exist
    When a 9th skill "plan-compliance" is added
    And the Planner runs
    Then 9 skills are dispatched

  Scenario: Spec status validation
    Given a spec with status "Draft"
    When the Planner is invoked
    Then it refuses to proceed
    And recommends handing off to the Spec Architect
```

### 11.3 Integration Tests

- Full pipeline: Spec Architect V2 produces spec -> Planner V2 decomposes -> Coder V2 consumes contracts.
- Auto-loop: Planner invokes Spec Architect subagent, spec is updated, Planner re-reads successfully.

### 11.4 End-to-End Tests

- Manual test: provide a real validated spec and run the Planner V2 through completion. Verify all outputs.

### 11.5 Performance Tests

- Time the full 8-skill dispatch on a medium-complexity spec. Target: under 60 minutes.
- Time the auto-loop with a spec needing 2 iterations of gap resolution.

### 11.6 Security Tests

- Verify no credentials appear in plan or contract files.
- Verify contract files contain only type definitions, no executable I/O code.

---

## 12. Constraints & Assumptions

### Constraints

- Must operate within the VS Code Copilot Chat agent framework.
- Skills execute sequentially (framework limitation).
- Context window limits require 800-line block cap for large contracts.
- All artifacts are local files; no external database.
- Auto-loop to Spec Architect requires runSubagent to support agent-to-agent invocation.

### Assumptions

1. The Spec Architect V2 (Spec 002) is implemented and can be invoked as a subagent for gap resolution.
2. Spec companion artifacts exist in `.sdd/specs/artifacts/` and are valid.
3. The target language is specified in the spec's technology stack section.
4. 800-line blocks are sufficient for per-WP contract generation.
5. The runSubagent tool supports file read/write within the subagent context.

---

## 13. Out of Scope

- **Code implementation**: The Planner produces plans and contracts, not executable code.
- **Spec writing**: Spec creation is the Spec Architect's responsibility. The Planner only triggers gap resolution.
- **Review of plans**: Plan review is done by the human at the approval gate.
- **Runtime deployment**: No deployment or infrastructure concerns.
- **Multi-language contracts**: Each plan targets one language. Multi-language projects need separate contract sets.
- **Contract file execution**: Contract files are type definitions only, never executed directly.

---

## 14. Open Questions

None remaining. All questions have been resolved:
1. Two-phase approach for context limits (resolved in FR-012).
2. Planner regenerates only affected contracts on spec revision (resolved in brief).
3. 800-line block limit per WP (resolved in FR-013).
4. Auto-loop max 3 iterations (resolved in FR-006).

---

## 15. Glossary

- **Auto-loop**: Autonomous invocation of the Spec Architect by the Planner to resolve spec gaps without human intervention.
- **Contract file**: A language-specific source file containing type definitions, interfaces, schemas, or constants that the Coder implements against.
- **Gap report**: A structured document listing spec incompleteness items discovered during the planning pre-check.
- **Phase 1**: Plan decomposition -- identifying WPs, tasks, dependencies, acceptance criteria.
- **Phase 2**: Contract generation -- producing language-specific contract files scoped per WP.
- **Plan accumulator**: The collection of WP files and README that skills read and append to during Phase 1.
- **WP (Work Package)**: A cohesive group of 5-12 related tasks that deliver a meaningful, testable increment.

---

## 16. Traceability Matrix

| FR ID | Requirement Summary | User Story | Acceptance Scenario | Test Type | Test Section Ref |
|-------|-------------------|------------|--------------------|-----------|----|
| FR-001 | Spec selection from .sdd/specs/ | US-01 | Scenario 1 | BDD | 11.2 |
| FR-002 | Read spec + companion artifacts | US-01 | Scenario 1 | BDD | 11.2 |
| FR-003 | Spec status validation | US-01 | BDD Scenario 8 | BDD | 11.2 |
| FR-004 | Spec completeness pre-check | US-03 | Scenario 1 | BDD | 11.2 |
| FR-005 | Verify spec artifact consistency | US-03 | Scenario 1 | BDD | 11.2 |
| FR-006 | Auto-loop to Spec Architect | US-03 | Scenario 1, 2, 3, BDD Scenario 2, 3 | BDD | 11.2 |
| FR-007 | Workspace research subagent | US-01 | Scenario 1 | BDD | 11.2 |
| FR-008 | Web research | US-01 | Scenario 1 | BDD | 11.2 |
| FR-009 | Patterns consumption | US-01 | Scenario 1 | BDD | 11.2 |
| FR-010 | Dynamic skill discovery | US-05 | Scenario 1, 2, BDD Scenario 7 | BDD | 11.2 |
| FR-011 | Deterministic skill ordering | US-01 | Scenario 1 | BDD | 11.2 |
| FR-012 | Two-phase execution | US-01, US-02 | Scenario 1 | BDD | 11.2 |
| FR-013 | 800-line block limit | US-02 | Scenario 1 | BDD | 11.2 |
| FR-014 | Skill dispatch via runSubagent | US-01 | Scenario 1, BDD Scenario 6 | BDD | 11.2 |
| FR-015 | Phase 1 sequential, Phase 2 skip-on-fail | US-01, US-02 | BDD Scenario 6 | BDD | 11.2 |
| FR-016 | Read plan state before writing | US-01 | Scenario 1 | BDD | 11.2 |
| FR-017 | Plan accumulator initialization | US-01 | Scenario 1 | BDD | 11.2 |
| FR-018 | Cross-WP consistency audit | US-04 | Scenario 1, 2, 3, BDD Scenario 4 | BDD | 11.2 |
| FR-019 | WP implementation-completeness check | US-01 | Scenario 1 | BDD | 11.2 |
| FR-020 | Present plan to user | US-01 | Scenario 1 | BDD | 11.2 |
| FR-021 | User feedback handling | US-01 | Scenario 1 | BDD | 11.2 |
| FR-022 | Commit policy | US-01 | Scenario 1 | BDD | 11.2 |
| FR-023 | Skill input contract (9 inputs) | US-05 | Scenario 1 | BDD | 11.2 |
| FR-024 | Skill execution sequence | US-05 | Scenario 1 | BDD | 11.2 |
| FR-025 | Phase 1 output to .sdd/plans/ | US-01 | Scenario 1 | BDD | 11.2 |
| FR-026 | Phase 2 output to contracts/ | US-02 | Scenario 1 | BDD | 11.2 |
| FR-027 | No modification of prior skill files | US-04 | Scenario 1 | BDD | 11.2 |
| FR-028 | Decomposition skill | US-01 | Scenario 1, 2, 3 | BDD | 11.2 |
| FR-029 | Task field requirements | US-01 | Scenario 1 | BDD | 11.2 |
| FR-030 | 5-12 tasks per WP target | US-01 | Scenario 1 | BDD | 11.2 |
| FR-031 | WP metadata table header | US-01 | Scenario 1 | BDD | 11.2 |
| FR-032 | Foundation WP virtual env setup | US-01 | Scenario 2 | BDD | 11.2 |
| FR-033 | Acceptance criteria skill | US-01 | Scenario 1 | BDD | 11.2 |
| FR-034 | FR traceability verification | US-01, US-04 | Scenario 1 | BDD | 11.2 |
| FR-035 | BDD/TDD requirements in tasks | US-01 | Scenario 1 | BDD | 11.2 |
| FR-036 | README updates by acceptance skill | US-01 | Scenario 1 | BDD | 11.2 |
| FR-037 | Interface contracts generation | US-02 | Scenario 1 | BDD | 11.2 |
| FR-038 | Interface match spec artifacts | US-02 | Scenario 1 | BDD | 11.2 |
| FR-039 | Contract manifest header | US-02 | Scenario 1 | BDD | 11.2 |
| FR-040 | Data schemas generation | US-02 | Scenario 1, BDD Scenario 5 | BDD | 11.2 |
| FR-041 | Data schemas match spec artifacts | US-02 | Scenario 1 | BDD | 11.2 |
| FR-042 | Shared entity deduplication | US-02 | Scenario 3 | BDD | 11.2 |
| FR-043 | API contracts generation | US-02 | Scenario 1 | BDD | 11.2 |
| FR-044 | API contracts match spec artifacts | US-02 | Scenario 1 | BDD | 11.2 |
| FR-045 | Error response types per endpoint | US-02 | Scenario 1 | BDD | 11.2 |
| FR-046 | State machine generation | US-02 | Scenario 1 | BDD | 11.2 |
| FR-047 | State machines match spec artifacts | US-02 | Scenario 1 | BDD | 11.2 |
| FR-048 | Error catalog generation | US-02 | Scenario 1 | BDD | 11.2 |
| FR-049 | Error catalogs match spec artifacts | US-02 | Scenario 1 | BDD | 11.2 |
| FR-050 | Shared error code deduplication | US-02 | Scenario 3 | BDD | 11.2 |
| FR-051 | Cross-WP validation skill | US-04 | Scenario 1, 2, 3, BDD Scenario 4 | BDD | 11.2 |
| FR-052 | Config schema generation | US-04 | Scenario 1 | BDD | 11.2 |
| FR-053 | Consistency fix documentation | US-04 | Scenario 1, BDD Scenario 4 | BDD | 11.2 |
| FR-054 | 100% spec artifact coverage | US-02, US-04 | Scenario 1 | BDD | 11.2 |

---

## 17. Technical References

### Architecture & Patterns
- Work Breakdown Structure best practices, PMI, consulted 2026-04-05
- Dependency graph analysis, https://en.wikipedia.org/wiki/Dependency_graph, consulted 2026-04-05

### Technology Stack
- VS Code Copilot Chat agents documentation, https://code.visualstudio.com/docs/copilot, consulted 2026-04-05

### Contract-First Design
- API-First Approach, https://swagger.io/resources/articles/adopting-an-api-first-approach/, consulted 2026-04-05
- Design by Contract, https://en.wikipedia.org/wiki/Design_by_contract, consulted 2026-04-05

### Standards & Specifications
- RFC 2119 Key Words, https://www.rfc-editor.org/rfc/rfc2119, consulted 2026-04-05

---

## 18. Version History

| Version | Date | Author | Summary of Changes |
|---------|------|--------|--------------------|
| 1.0 | 2026-04-05 | Spec Architect | Initial specification |
| 1.0.1 | 2026-04-05 | Spec Architect | Self-review corrections: ensured two-phase execution clearly documented, verified all FRs use SHALL, confirmed traceability matrix completeness, verified no ambiguous language |
| 1.1 | 2026-04-05 | Spec Architect | Validation: completed traceability matrix (26 missing FRs added, now 54/54), encoding verified clean, no ambiguous language, no unresolved markers. Status changed to Validated |
