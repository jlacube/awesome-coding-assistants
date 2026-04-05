---
skill: review-spec
wp: WP16-phase1-decomposition-acceptance
spec: .sdd/specs/003-planner-v2.spec.md
reviewed_at: 2026-04-05T13:00:00Z
status: completed
finding_counts:
  pass: 10
  warn: 0
  fail: 2
  na: 3
files_reviewed:
  - .github/skills/plan-decomposition/SKILL.md
  - .github/skills/plan-acceptance/SKILL.md
  - .github/skills/PLAN-SKILL-CONTRACT.md
  - .sdd/specs/003-planner-v2.spec.md
  - .sdd/plans/WP16-phase1-decomposition-acceptance.md
---

# review-spec Findings for WP16-phase1-decomposition-acceptance

## Summary

Evaluated 9 functional requirements (FR-028 through FR-036) across two implementation files: `plan-decomposition/SKILL.md` and `plan-acceptance/SKILL.md`. FR-033 was evaluated at the sub-item level (4 sub-items). Of 12 substantive checks, 10 are Compliant and 2 are Partial/Deviating for FR-033.3 (implementation guidance template). Three checklist categories are N/A for this WP (API contracts, error code returns, runtime success criteria).

## Findings

### SPEC-001 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-028
- **File**: .github/skills/plan-decomposition/SKILL.md
- **Description**: All 7 sub-items of FR-028 are fully implemented. Step 1 (L42) analyzes FRs, user stories, and architecture. Step 2 (L57) identifies WPs with the required sequencing logic (Foundation -> Core domain -> Integrations -> User-facing -> Quality -> Delivery) and assigns P0/P1/P2+ priorities. Step 3 (L83) decomposes WPs into 5-12 atomic tasks. Step 4 (L113) defines inter-task and inter-WP dependencies by ID. Step 5 (L129) writes WP files with the standard template. Step 6 (L181) writes the skeleton README with WP index and dependency graph.

### SPEC-002 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-029
- **File**: .github/skills/plan-decomposition/SKILL.md#L85-L103
- **Description**: The task template in Step 3 includes all 7 required fields: (1) unique T\<NN>-XX identifier, (2) Description, (3) Spec refs, (4) Parallel flag, (5) Placeholder for acceptance criteria (explicitly marked as filled by plan-acceptance), (6) Placeholder for implementation guidance (explicitly marked as filled by plan-acceptance), (7) Dependencies. The constraints section (L197) reinforces that acceptance criteria and implementation guidance placeholders must not be filled by this skill.

### SPEC-003 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-030
- **File**: .github/skills/plan-decomposition/SKILL.md#L75-L79
- **Description**: Step 2 splitting rules explicitly state "Target 5-12 tasks per WP (FR-030)" and provide actionable guidance: fewer than 5 means merge, more than 12 means split. The rule is cited by FR number for traceability.

### SPEC-004 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-031
- **File**: .github/skills/plan-decomposition/SKILL.md#L139-L149
- **Description**: The WP file template in Step 5 includes all 7 metadata table fields required by FR-031: Spec, Priority, Lane, Depends on, Goal, Status, Independent Test. The template also includes two additional fields (Parallelisable, Prompt) which are consistent with the Section 7.1 data model and do not conflict with the spec.

### SPEC-005 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-032
- **File**: .github/skills/plan-decomposition/SKILL.md#L105-L111
- **Description**: The "Foundation WP special rule (FR-032)" in Step 3 explicitly states: "The first task of the foundation WP SHALL always be virtual environment setup for languages with package isolation." Three concrete examples are provided (Python: venv/poetry/conda, Node.js: local node_modules with lockfile, Go: go modules) plus a fallback for languages without package isolation.

### SPEC-006 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-033.1
- **File**: .github/skills/plan-acceptance/SKILL.md#L59-L73
- **Description**: Step 2a "Extract SHALL Statements (FR-033.1)" correctly instructs the skill to copy exact SHALL/SHALL NOT statements as acceptance criterion checkboxes, enforces the minimum of 3 per task, and explicitly states "Use the spec wording verbatim -- do not paraphrase." The output format matches the expected checkbox markdown structure.

### SPEC-007 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-033.2
- **File**: .github/skills/plan-acceptance/SKILL.md#L75-L80
- **Description**: Step 2b "Map BDD Scenarios (FR-033.2)" correctly instructs the skill to copy Given/When/Then scenarios from the spec's test strategy section, map each scenario to the task's FR refs, and flag gaps where a FR has no BDD scenario.

### SPEC-008 [FAIL]
- **Checklist item**: FR classification - SHALL obligation (Partial)
- **Requirement**: FR-033.3
- **File**: .github/skills/plan-acceptance/SKILL.md#L90-L107
- **Description**: The implementation guidance template is missing two fields required by FR-033.3. The spec requires "official doc links, recommended patterns, known pitfalls, error codes, and validation rules" (5 items). The template provides only 4 items, substituting "Files to create/modify" for the missing "error codes" and "validation rules."
- **Expected**: Implementation guidance template with fields: Official docs, Patterns, Known pitfalls, Error codes, Validation rules (per FR-033 item 3: "Add implementation guidance with official doc links, recommended patterns, known pitfalls, error codes, and validation rules")
- **Evidence**:
  ```markdown
  ## Step 3 - Add Implementation Guidance (FR-033.4)
  
  - **Implementation Guidance**:
    - Official docs: <URL or reference for the primary library/API used>
    - Patterns: <Specific design pattern or approach from the spec/architecture>
    - Known pitfalls: <Common mistakes or gotchas for this feature area>
    - Files to create/modify: <Specific paths based on the spec's directory structure>
  ```
  Additionally, the heading labels this step as "FR-033.4" but the spec's FR-033 item 3 is implementation guidance (FR-033 item 4 is test requirements). Step 2c is labeled "FR-033.3" but covers error path criteria, which is not a numbered sub-item of FR-033. This cross-reference mismatch further obscures the gap.

### SPEC-009 [FAIL]
- **Checklist item**: Data model match
- **Requirement**: FR-033.3 / Section 7.1
- **File**: .github/skills/plan-acceptance/SKILL.md#L93-L99
- **Description**: Section 7.1 "Plan Accumulator Files" defines the Task's Implementation Guidance field with constraints "doc links, patterns, pitfalls, error codes". The plan-acceptance template omits "error codes" and includes "Files to create/modify" instead (which is not in the Section 7.1 constraints). This is the same underlying gap as SPEC-008 viewed through the data model lens.
- **Expected**: Implementation Guidance template fields matching Section 7.1 constraints: doc links, patterns, pitfalls, error codes
- **Evidence**:
  ```
  Spec Section 7.1 Task data model:
  | Implementation Guidance | object | doc links, patterns, pitfalls, error codes | Coder reference |
  
  Actual template fields in plan-acceptance Step 3:
  Official docs, Patterns, Known pitfalls, Files to create/modify
  ```

### SPEC-010 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-033.4
- **File**: .github/skills/plan-acceptance/SKILL.md#L109-L126
- **Description**: Step 4 "Set Test Requirements (FR-035)" covers FR-033.4 (test requirements per task) with a decision table mapping FR types to default test types (unit, integration, BDD, E2E, none). Although the heading cites FR-035 instead of FR-033.4, the behavior satisfies the FR-033 item 4 obligation.

### SPEC-011 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-034
- **File**: .github/skills/plan-acceptance/SKILL.md#L130-L159
- **Description**: Step 5 "Verify FR Traceability (FR-034)" implements comprehensive traceability with four checks: forward trace (FR -> Task), backward trace (Task -> FR), completeness (no unassigned FRs), and uniqueness (no duplicate assignments). Gap handling correctly flags unassigned FRs as `[GAP]` and untraced tasks as `[UNTRACED]`. A per-WP traceability table output format is defined.

### SPEC-012 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-035
- **File**: .github/skills/plan-acceptance/SKILL.md#L119-L126
- **Description**: The BDD/TDD mandate section explicitly states: specs with BDD scenarios MUST have matching BDD test types, every task with test requirements MUST follow TDD (test scenarios before implementation), and coverage thresholds from the spec must be noted. This directly satisfies FR-035's requirement that "tests derive from spec acceptance scenarios, not from implementation."

### SPEC-013 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-036
- **File**: .github/skills/plan-acceptance/SKILL.md#L163-L175
- **Description**: Step 6 "Update README (FR-036)" covers all 3 required items: (1) WP index table with columns WP ID, Title, Priority, Lane, Depends On, Tasks Count, FR Count; (2) MVP scope section listing P0/P1 WPs as MVP vs P2+ as post-MVP; (3) Dependency graph in mermaid format. An additional FR coverage summary is included which exceeds spec requirements.

### SPEC-014 [N/A]
- **Checklist item**: API contract match
- **Justification**: No API endpoints in this WP. Both plan-decomposition and plan-acceptance are LLM instruction files (SKILL.md) that produce markdown artifacts, not runtime code with API surfaces. Section 8 API contracts do not apply.

### SPEC-015 [N/A]
- **Checklist item**: Error codes returned
- **Justification**: These are LLM skill instruction files, not executable code. They do not return error codes at runtime. Error handling within the skills is expressed as gap-flagging instructions (e.g., `[GAP]`, `[UNTRACED]` markers) rather than programmatic error returns.

### SPEC-016 [N/A]
- **Checklist item**: Success criteria verification
- **Justification**: Deferred verification: the WP's Independent Test ("Dispatch plan-decomposition and plan-acceptance against a validated spec") requires runtime execution of the planner coordinator dispatching these skills as subagents. This cannot be verified by static review of the SKILL.md instruction files alone.
