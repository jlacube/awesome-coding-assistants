---
lane: for_review
---

# WP25 - review-spec-completeness Skill

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/005-review-spec-completeness.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | none |
| Goal | Create the review-spec-completeness skill that validates spec implementation-readiness before the Planner decomposes it |
| Status | Not Started |
| Independent Test | Install the skill and invoke the Review Coordinator (or Planner pre-check) on a spec missing error behaviors for 3 FRs. Verify: 3 HIGH findings for missing error behaviors, verdict FAIL |
| Parallelisable | Yes (with WP26) |
| Prompt | `.sdd/plans/WP25-review-spec-completeness.md` |

## Objective

Create `.github/skills/review-spec-completeness/SKILL.md` -- a pre-planning validation gate that checks specifications for implementation completeness. The skill runs 10 distinct checks (obligation language, error behavior, data model, API contracts, state machines, traceability, integrations, ambiguity, artifact consistency, security) and produces structured findings with a PASS/FAIL verdict. This prevents the Planner from decomposing shallow specs that cause downstream rework.

## Spec References

FR-001, FR-002, FR-003, FR-004, FR-005, FR-006, FR-007, FR-008, FR-009, FR-010, FR-011, FR-012, FR-013, FR-014, Section 5 US-01, Section 7.1 (Completeness Finding), Section 7.3 (Verdict), Section 8.2 (Dispatch Prompt), Section 9.3 (Directory Structure), Section 11.2 (BDD Scenarios)

## Tasks

### T25-01 - Create review-spec-completeness directory and SKILL.md with YAML frontmatter

- **Description**: Create the directory `.github/skills/review-spec-completeness/` and the `SKILL.md` file. Write the YAML frontmatter (name, description, argument-hint) and the purpose/invocation overview explaining when and how this skill is dispatched.
- **Spec refs**: FR-001, FR-002, Section 7.5, Section 9.3
- **Parallel**: No (foundation for all T25 tasks)
- **Acceptance criteria**:
  - [x] File exists at `.github/skills/review-spec-completeness/SKILL.md`
  - [x] YAML frontmatter `name` is `review-spec-completeness`
  - [x] YAML frontmatter `description` explains the skill validates spec completeness before planning
  - [x] Purpose section states the skill SHALL validate that a specification is implementation-complete before planning begins (FR-001)
  - [x] Input contract states the skill SHALL read the spec file and its companion artifacts directory as inputs (FR-002)
  - [x] The skill is discoverable via the Review Coordinator's `review-*/SKILL.md` glob pattern (SC-003)
- **Test requirements**: none (structural verification)
- **Depends on**: none
- **Implementation Guidance**:
  - Follow the existing review skill YAML frontmatter pattern from `.github/skills/review-spec/SKILL.md`:
    ```yaml
    ---
    name: review-spec-completeness
    description: "Pre-planning spec completeness validation. Checks obligation language, error behaviors, data model depth, API contracts, state machines, traceability, integrations, artifact consistency, and security requirements."
    argument-hint: "Invoked by Review Coordinator or Planner pre-check - do not call directly"
    ---
    ```
  - The skill file body is natural language instructions that a subagent reads and follows
  - Document both invocation paths: dispatched by Review Coordinator OR directly by the Planner coordinator during its completeness pre-check
  - Error handling: Spec file not found SHALL halt with error report; artifacts directory not found SHALL flag as HIGH finding but not halt

### T25-02 - Write finding format and verdict output sections

- **Description**: Write the output format instructions defining the finding structure (SPEC-COMP-XXX format) and verdict determination (PASS when 0 HIGH findings, FAIL when 1+ HIGH findings).
- **Spec refs**: FR-013, FR-014, Section 7.1 (Completeness Finding), Section 7.3 (Verdict)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Finding format matches FR-013 exactly: id (SPEC-COMP-XXX), severity (HIGH/MEDIUM/LOW), category (10 categories from FR-013), location, issue, recommendation
  - [x] All 10 categories are listed: obligation-language, error-behavior, data-model, api-contract, state-machine, traceability, integration, ambiguity, artifact-consistency, security
  - [x] Verdict rule states PASS when zero HIGH findings and FAIL when one or more HIGH findings (FR-014)
  - [x] Output includes finding counts by severity (high_count, medium_count, low_count)
  - [x] Finding field constraints match Section 7.1: issue and recommendation are 1-500 chars
- **Test requirements**: BDD - Section 11.2 "Pass a complete spec" scenario
- **Depends on**: T25-01
- **Implementation Guidance**:
  - Use the data model from `.sdd/specs/artifacts/005-review-spec-completeness/data-models.ts` as the canonical field reference
  - The finding format aligns with the review coordinator's standard finding aggregation
  - Include the exact finding template from FR-013 in the SKILL.md so the subagent can copy it

### T25-03 - Write obligation language and ambiguity checks

- **Description**: Write the checklist sections for: (1) checking that every FR uses SHALL/SHALL NOT obligation language, and (2) checking for ambiguous language in FR and NFR text.
- **Spec refs**: FR-003, FR-010
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] SHALL language check instructs: every FR using "should", "could", "might", "may", or "can" (as obligation, not permission) SHALL be flagged with severity HIGH, category obligation-language (FR-003)
  - [x] Ambiguity check instructs: any occurrence of "appropriate", "reasonable", "as needed", "etc.", "similar", "relevant" in FR or NFR text SHALL be flagged with severity MEDIUM, category ambiguity (FR-010)
  - [x] Both checks specify the exact words to scan for
  - [x] BDD scenario covered: "Given a spec where FR-003 uses 'should', When review-spec-completeness runs, Then finding SPEC-COMP-001 is reported with severity HIGH and category obligation-language"
- **Test requirements**: BDD - Section 11.2 "Flag FRs with weak obligation language" scenario
- **Depends on**: T25-02
- **Implementation Guidance**:
  - These are text-pattern scanning checks -- the subagent should scan every FR-XXX and NFR-XXX statement
  - Distinguish "may" as permission (acceptable) vs "may" as obligation (flag it). Example: "Users may optionally..." is permission; "The system may handle..." is obligation
  - Group these together since both involve scanning FR/NFR text for specific word patterns

### T25-04 - Write error behavior check

- **Description**: Write the checklist section that verifies every FR has defined error behavior (what happens when the happy path fails).
- **Spec refs**: FR-004
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Check instructs: every FR SHALL have defined error behavior (FR-004)
  - [x] FRs without error paths SHALL be flagged with severity HIGH, category error-behavior
  - [x] The recommendation SHALL suggest adding error behavior
  - [x] BDD scenario covered: "Given a spec where FR-005 has no error behavior defined, When the skill runs, Then a HIGH finding for missing error behavior is reported and the recommendation suggests adding error behavior"
- **Test requirements**: BDD - Section 11.2 "Flag missing error behavior" scenario
- **Depends on**: T25-02
- **Implementation Guidance**:
  - Error behavior is typically found as "Error:" bullets under an FR, or in a separate "Error behaviors" subsection
  - The check should look for each FR's failure/error path, not just the happy path
  - Common patterns for error behavior: "If X fails, then Y", "When X is invalid, the system SHALL...", "Error: ..."

### T25-05 - Write data model completeness check

- **Description**: Write the checklist section that verifies every entity in the data model has all fields with explicit types, nullability, constraints, validation rules, and default values.
- **Spec refs**: FR-005, Section 7 Data Model
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Check instructs: every entity in Section 7 SHALL have all 5 properties per field (FR-005):
    1. All fields with explicit types (no untyped fields)
    2. Nullability declared for every field
    3. Constraints (required, unique, max length, format, min/max)
    4. Validation rules beyond type constraints
    5. Default values (or explicit "no default")
  - [x] Missing items SHALL be flagged with severity HIGH, category data-model
  - [x] BDD scenario covered: "Given a spec where entity User has field 'role' with no type, When the skill runs, Then a HIGH finding for untyped field is reported"
- **Test requirements**: BDD - Section 11.2 "Flag incomplete data model" scenario
- **Depends on**: T25-02
- **Implementation Guidance**:
  - The subagent should iterate over every entity table in Section 7
  - Each row in the entity table represents a field; check all 5 properties per field
  - "No default" is acceptable as an explicit declaration -- the absence of any default mention is what triggers a finding

### T25-06 - Write API endpoint completeness check

- **Description**: Write the checklist section that verifies every API endpoint has all applicable HTTP error codes, typed request/response schemas, and auth requirements.
- **Spec refs**: FR-006, Section 8 API/Interface Design
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Check instructs: every API endpoint in Section 8 SHALL have all 4 properties (FR-006):
    1. All applicable HTTP error codes (400, 401, 403, 404, 409, 422, 500) with meanings and response bodies
    2. Request schema with all fields typed
    3. Response schema with all fields typed
    4. Auth requirements stated
  - [x] Missing items SHALL be flagged with severity HIGH, category api-contract
  - [x] The check lists all 7 HTTP error codes explicitly (400, 401, 403, 404, 409, 422, 500)
- **Test requirements**: BDD - Section 11.2 "Flag missing error behavior" scenario (API variant)
- **Depends on**: T25-02
- **Implementation Guidance**:
  - Not every endpoint will use all 7 error codes (e.g., a GET endpoint may not have 409 Conflict)
  - The check should verify that "all applicable" codes are present -- the subagent must judge applicability based on the endpoint's HTTP method and semantics
  - For specs without HTTP APIs (e.g., skill-based dispatch), this check may produce N/A findings -- that is acceptable

### T25-07 - Write state machine completeness check

- **Description**: Write the checklist section that verifies every entity with a status/state field has all valid states, transitions, guards, and side effects defined.
- **Spec refs**: FR-007
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Check instructs: every entity with a status/state field SHALL have all 4 properties (FR-007):
    1. All valid states listed as an enum
    2. All valid transitions defined (from-state, to-state)
    3. Guards/conditions on each transition
    4. Side effects per transition
  - [x] Missing items SHALL be flagged with severity MEDIUM, category state-machine
  - [x] Note: severity is MEDIUM (not HIGH) per FR-007
- **Test requirements**: BDD - Section 11.2 "Pass a complete spec" scenario (absence of state machine findings)
- **Depends on**: T25-02
- **Implementation Guidance**:
  - State fields are identified by names like "status", "state", "phase", "stage", or enum fields with lifecycle semantics
  - Not every entity has a state field -- the check only applies when one is found
  - If no state fields exist in the spec, no findings are produced for this category

### T25-08 - Write traceability matrix check

- **Description**: Write the checklist section that verifies the traceability matrix (Section 16) has no empty cells: every FR maps to a US, every US maps to scenarios, scenarios map to test types.
- **Spec refs**: FR-008, Section 16 Traceability Matrix
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Check instructs: the traceability matrix SHALL have no empty cells (FR-008):
    1. Every FR maps to at least one US
    2. Every US maps to at least one acceptance scenario
    3. Every acceptance scenario maps to at least one test type
    4. Every test type maps to a test section reference
  - [x] Empty cells SHALL be flagged with severity HIGH, category traceability
  - [x] BDD scenario covered: "Given Section 16 has FR-012 with no US mapping, When the skill runs, Then a HIGH finding for empty traceability cell is reported"
- **Test requirements**: BDD - Section 11.2 "Flag empty traceability cell" scenario
- **Depends on**: T25-02
- **Implementation Guidance**:
  - The traceability matrix is typically a table in Section 16 with columns: FR ID, Requirement Summary, User Story, Acceptance Scenario, Test Type, Test Section Ref
  - Check each row; any cell that is empty or contains only whitespace is flagged
  - If Section 16 does not exist, flag the entire section as missing (HIGH finding)

### T25-09 - Write integration strategy and security requirements checks

- **Description**: Write the checklist sections for: (1) verifying every external integration has timeout, retry, fallback, and circuit breaker defined, and (2) verifying security requirements have per-component detail, OWASP references, and data sensitivity classification.
- **Spec refs**: FR-009, FR-012, Section 9.5, Section 10.2
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Integration check instructs: every external integration in Section 9.5 SHALL have timeout, retry strategy, fallback behavior, and circuit breaker threshold if applicable (FR-009)
  - [x] Missing integration items SHALL be flagged with severity MEDIUM, category integration
  - [x] Security check instructs: security requirements SHALL include per-component security requirements, OWASP mitigation references, and data sensitivity classification per entity (FR-012)
  - [x] Missing security items SHALL be flagged with severity MEDIUM, category security
  - [x] If no external integrations exist, integration check produces no findings (N/A)
- **Test requirements**: BDD - Section 11.2 "Pass a complete spec" scenario (all checks passing)
- **Depends on**: T25-02
- **Implementation Guidance**:
  - These checks share severity level (MEDIUM) and both verify existence of specific subsections
  - Integration section may not exist in specs with no external dependencies -- that is acceptable, not a gap
  - Security check: "per-component" means each major module/service has its own security considerations, not just a blanket "follow OWASP" statement
  - Circuit breaker is "if applicable" -- only flag if the integration pattern warrants it (high-volume, critical path)

### T25-10 - Write artifact consistency check

- **Description**: Write the checklist section that verifies companion artifact files exist in `.sdd/specs/artifacts/` and are consistent with the prose spec: entity types match, API types match, error codes match, field names and types match between prose and artifacts.
- **Spec refs**: FR-011, Section 7, Section 8, Section 4
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Check instructs: companion artifacts SHALL exist and be consistent with the prose spec (FR-011):
    1. Every entity in Section 7 has a corresponding type definition in `data-models.<ext>`
    2. Every API endpoint in Section 8 has corresponding request/response types in `api-contracts.<ext>`
    3. Every error code in Section 4 has a corresponding entry in `error-catalog.<ext>`
    4. Field names and types match between prose and artifacts
  - [x] Missing or inconsistent artifacts SHALL be flagged with severity HIGH, category artifact-consistency
  - [x] If the artifacts directory does not exist, flag as HIGH finding (artifacts expected for V2 specs) but do not halt
- **Test requirements**: BDD - Section 11.2 "Pass a complete spec" scenario
- **Depends on**: T25-02
- **Implementation Guidance**:
  - Artifact files use language-appropriate extensions: `.ts` for TypeScript, `.py` for Python, etc.
  - "Consistent" means: field names are identical (case-sensitive), types match semantically (e.g., `string` in prose matches `string` in TypeScript)
  - The check should cross-reference each entity/endpoint/error code between the prose and the artifact file
  - Not all specs will have all artifact types -- flag what is missing but available in the prose

### T25-11 - Integration verification with Review Coordinator

- **Description**: Verify the skill integrates with the Review Coordinator's dynamic discovery mechanism. The coordinator discovers skills via glob pattern `review-*/SKILL.md` and dispatches them as subagents.
- **Spec refs**: SC-003, Section 9.1
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill directory name `review-spec-completeness` matches the Review Coordinator's glob pattern `review-*/SKILL.md`
  - [x] The skill's YAML frontmatter is parseable by the coordinator
  - [x] No Review Coordinator changes are needed to discover and dispatch this skill (SC-003)
  - [x] The dispatch prompt from Section 8.2 is compatible with the skill's input contract
- **Test requirements**: BDD - manual invocation via coordinator
- **Depends on**: T25-01 through T25-10
- **Implementation Guidance**:
  - The Review Coordinator at `.github/agents/reviewer.agent.md` uses glob `review-*/SKILL.md` for discovery
  - Verify by checking that `review-spec-completeness/SKILL.md` matches the glob
  - Verify that the dispatch prompt from spec Section 8.2 provides all inputs the skill expects
  - No changes to the coordinator agent file should be needed

## Implementation Notes

- All artifacts are markdown files. There is no executable code, build system, or test framework.
- "Testing" means manually invoking the skill (via coordinator or directly) against a spec with known issues and verifying the findings output matches the BDD scenarios.
- The skill follows the same file structure pattern as existing review skills (review-spec, review-security, review-quality).
- The 10 completeness checks (FR-003 through FR-012) should be written as clearly delimited sections in the SKILL.md so the subagent can follow them sequentially.
- Tasks T25-03 through T25-10 are parallelizable since they each write independent sections of the same SKILL.md file. In practice, they will be applied sequentially to avoid merge conflicts.

## Parallel Opportunities

Tasks T25-03 through T25-10 write independent checklist sections and can conceptually be worked in parallel. T25-01 and T25-02 must come first (file creation and output format). T25-11 must come last (integration verification).

## Risks & Mitigations

- **Risk**: Checklist sections may be too prescriptive, causing the subagent to apply checks mechanically without judgment. **Mitigation**: Include guidance on when checks are N/A (e.g., no external integrations means no integration check findings).
- **Risk**: Ambiguity detection (FR-010) may produce false positives for legitimate uses of flagged words. **Mitigation**: Scope the check to FR and NFR text only, not prose descriptions or examples.

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-06T00:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-06T00:00:00Z - coder - T25-01 - completed - Created SKILL.md with YAML frontmatter and purpose section
- 2026-04-06T00:00:00Z - coder - T25-02 - completed - Finding format (SPEC-COMP-XXX) and verdict (PASS/FAIL) sections
- 2026-04-06T00:00:00Z - coder - T25-03 - completed - Obligation language (Check 1) and ambiguity (Check 7) sections
- 2026-04-06T00:00:00Z - coder - T25-04 - completed - Error behavior check (Check 2)
- 2026-04-06T00:00:00Z - coder - T25-05 - completed - Data model completeness check (Check 3)
- 2026-04-06T00:00:00Z - coder - T25-06 - completed - API endpoint completeness check (Check 4)
- 2026-04-06T00:00:00Z - coder - T25-07 - completed - State machine completeness check (Check 5)
- 2026-04-06T00:00:00Z - coder - T25-08 - completed - Traceability matrix check (Check 6)
- 2026-04-06T00:00:00Z - coder - T25-09 - completed - Integration strategy (Check 8) and security requirements (Check 10)
- 2026-04-06T00:00:00Z - coder - T25-10 - completed - Artifact consistency check (Check 9)
- 2026-04-06T00:00:00Z - coder - T25-11 - completed - Integration verification with Review Coordinator glob pattern
- 2026-04-06T00:00:00Z - coder - lane=for_review - All tasks complete, all acceptance criteria met
