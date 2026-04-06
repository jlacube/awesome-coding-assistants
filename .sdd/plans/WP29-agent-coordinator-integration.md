---
lane: done
---

# WP29 - Agent Coordinator Integration

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/006-handoff-schemas-patterns.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP27, WP28 |
| Goal | All agent coordinators validate handoffs against schemas and consume domain-specific patterns; Review Coordinator curates patterns automatically |
| Status | Complete |
| Independent Test | Attempt a handoff to the Planner with a Draft spec; verify schema validation blocks it. Start the Spec Architect; verify only spec-patterns are loaded. Submit 3 reviews with the same finding; verify a new pattern is created. |
| Parallelisable | No |
| Prompt | `.sdd/plans/WP29-agent-coordinator-integration.md` |

## Objective

Wire the handoff schemas (WP27) and domain-specific pattern files (WP28) into the existing agent coordinator files. Each coordinator validates incoming handoffs against the relevant schema before any other action, reads its domain-specific patterns at startup, and the Review Coordinator gains automated pattern curation and retirement capabilities.

## Spec References

FR-004, FR-005, FR-011, FR-012, FR-013, FR-014, FR-015, Section 6.1, Section 6.2, US-01, US-02, US-03

## Tasks

### T29-01 - Add schema validation step to Spec Architect coordinator

- **Description**: Update `.github/agents/spec-architect.agent.md` to validate incoming handoff against the relevant schema file (`ideation-to-spec.schema.yaml` or `reviewer-to-spec.schema.yaml`) as the FIRST action before any research or skill dispatch. The coordinator SHALL check required_artifacts, required_state, context_fields, and validation_rules from the schema.
- **Spec refs**: FR-004, FR-005
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Spec Architect coordinator SHALL validate incoming handoff against the relevant schema before proceeding (FR-004)
  - [x] Schema validation SHALL be the FIRST action performed after receiving a handoff, before any research or skill dispatch (FR-005)
  - [x] Validation SHALL check required_artifacts exist and pass their validation rules (FR-004.1)
  - [x] Validation SHALL check required_state conditions are met (FR-004.2)
  - [x] Validation SHALL check context_fields are present and have valid values (FR-004.3)
  - [x] If any validation fails, the coordinator SHALL halt and report which checks failed with the schema's error messages (FR-004 error)
- **Test requirements**: BDD
- **Depends on**: none
- **Implementation Guidance**:
  - Add a new "Schema Validation" step at the very beginning of the coordinator workflow, before Step 1
  - Read the schema file path from `.github/schemas/ideation-to-spec.schema.yaml` or `.github/schemas/reviewer-to-spec.schema.yaml` depending on the handoff source
  - Implement validation as sequential checks: (1) required_artifacts exist, (2) required_state conditions true, (3) context_fields present, (4) validation_rules pass
  - On failure: halt with clear error listing which checks failed and the error messages from the schema
  - Files to modify: `.github/agents/spec-architect.agent.md`
  - Known pitfall: must determine which schema to use based on the source agent in the handoff context

### T29-02 - Add schema validation step to Planner coordinator

- **Description**: Update `.github/agents/planner.agent.md` (the Planner mode instructions) to validate incoming handoff against `spec-to-planner.schema.yaml` as the FIRST action. Validation includes checking spec status is "Validated" and companion artifacts exist.
- **Spec refs**: FR-004, FR-005
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Planner coordinator SHALL validate incoming handoff against `spec-to-planner.schema.yaml` before proceeding (FR-004)
  - [x] Schema validation SHALL be the FIRST action, before spec selection or any research (FR-005)
  - [x] Validation SHALL verify spec file exists and has Status: Validated
  - [x] Validation SHALL verify companion artifacts directory exists with at least 1 file
  - [x] If validation fails, the coordinator SHALL halt with error "Spec must be Validated before planning" (FR-003 example)
- **Test requirements**: BDD (Scenario: Invalid handoff fails schema - Draft spec)
- **Depends on**: none
- **Implementation Guidance**:
  - The Planner already checks spec status in Step 1 -- schema validation formalizes this as a schema-driven check
  - Add schema validation as Step 0 before the existing Step 1
  - Read `.github/schemas/spec-to-planner.schema.yaml` and execute its validation rules
  - The planner.agent.md may need to be updated in the mode instructions section
  - Files to modify: `.github/agents/planner.agent.md`

### T29-03 - Add schema validation step to Coder coordinator

- **Description**: Update `.github/agents/coder.agent.md` to validate incoming handoff against `planner-to-coder.schema.yaml` or `reviewer-to-coder.schema.yaml` as the FIRST action. Validation includes checking WP file exists and plan artifacts are present.
- **Spec refs**: FR-004, FR-005
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Coder coordinator SHALL validate incoming handoff against the relevant schema before proceeding (FR-004)
  - [x] Schema validation SHALL be the FIRST action, before WP selection or artifact loading (FR-005)
  - [x] Validation SHALL verify WP file exists
  - [x] Validation SHALL verify spec and contracts context is provided
  - [x] If validation fails, the coordinator SHALL halt and report failed checks (FR-004 error)
- **Test requirements**: BDD
- **Depends on**: none
- **Implementation Guidance**:
  - Determine which schema to use: `planner-to-coder.schema.yaml` for fresh implementation, `reviewer-to-coder.schema.yaml` for rework
  - Add schema validation as the first step in the coder coordinator workflow
  - Files to modify: `.github/agents/coder.agent.md`

### T29-04 - Add schema validation step to Review Coordinator

- **Description**: Update `.github/agents/review-coordinator.agent.md` to validate incoming handoff against `coder-to-reviewer.schema.yaml` as the FIRST action. Validation includes checking implementation files exist and WP has implementation data.
- **Spec refs**: FR-004, FR-005
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Review Coordinator SHALL validate incoming handoff against `coder-to-reviewer.schema.yaml` before proceeding (FR-004)
  - [x] Schema validation SHALL be the FIRST action, before scope selection or artifact loading (FR-005)
  - [x] Validation SHALL verify implementation artifacts exist
  - [x] If validation fails with missing implementation, the coordinator SHALL report "missing implementation" error (US-01 Scenario 3)
- **Test requirements**: BDD (Scenario: Invalid handoff fails schema - missing implementation)
- **Depends on**: none
- **Implementation Guidance**:
  - Add schema validation before the existing Step 1 (scope selection)
  - Read `.github/schemas/coder-to-reviewer.schema.yaml` and execute its checks
  - Files to modify: `.github/agents/review-coordinator.agent.md`

### T29-05 - Formalize domain-specific pattern consumption in all coordinators

- **Description**: Ensure each agent coordinator reads ONLY its domain-specific patterns file at startup before any skill dispatch. Spec Architect reads only `spec-patterns.md`, Planner reads only `plan-patterns.md`, Coder reads only `code-patterns.md`. Verify no cross-domain pattern inclusion occurs.
- **Spec refs**: FR-011, FR-012
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Each agent coordinator SHALL read its domain-specific patterns file at startup before any skill dispatch (FR-011)
  - [x] Active patterns from the domain file SHALL be included in the prompt for every skill that agent dispatches (FR-011)
  - [x] Spec Architect reads ONLY `spec-patterns.md`; Planner reads ONLY `plan-patterns.md`; Coder reads ONLY `code-patterns.md` (FR-012)
  - [x] If the patterns file does not exist, the agent SHALL proceed without patterns and log a warning (FR-008 error, FR-011 error)
  - [x] If cross-domain patterns are detected in an agent's prompt, the coordinator SHALL strip them before skill dispatch (FR-012 error)
- **Test requirements**: BDD (Scenario: Agent reads domain patterns at startup; Scenario: Agent ignores other domain patterns)
- **Depends on**: none
- **Implementation Guidance**:
  - Spec Architect already reads spec-patterns.md (line 131 of spec-architect.agent.md) -- verify it follows FR-011 exactly
  - Planner already reads plan-patterns.md (Step 5 in planner mode instructions) -- verify compliance
  - Coder already reads code-patterns.md (T21-04 from WP21) -- verify compliance
  - For each agent, verify: (1) reads at startup, (2) before skill dispatch, (3) only its domain file, (4) handles missing file gracefully
  - Files to modify: `.github/agents/spec-architect.agent.md`, `.github/agents/planner.agent.md`, `.github/agents/coder.agent.md`
  - Known pitfall: some agents may currently read the wrong patterns file or read patterns after skill dispatch

### T29-06 - Add automated pattern curation to Review Coordinator

- **Description**: Update the Review Coordinator to track finding recurrence across reviews. When the same finding category appears in 3 or more reviews, automatically create a new pattern entry in the relevant domain-specific file with status "active", trigger, prevention, and example from the recurring findings.
- **Spec refs**: FR-013
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The Review Coordinator SHALL track finding recurrence across reviews (FR-013)
  - [x] When the same finding category appears in 3 or more reviews, the coordinator SHALL create a new pattern entry in the relevant domain-specific file (FR-013.1)
  - [x] New patterns SHALL have status set to "active" (FR-013.2)
  - [x] New patterns SHALL include trigger, prevention, and example from the recurring findings (FR-013.3)
  - [x] If the coordinator cannot determine the target domain, the pattern SHALL be placed in the closest-matching domain file with a `[NEEDS REVIEW]` tag (FR-013 error)
- **Test requirements**: BDD (Scenario: Pattern curation on recurring finding)
- **Depends on**: T29-05
- **Implementation Guidance**:
  - The Review Coordinator currently has pattern curation (lines 364, 435) referencing legacy `review-patterns.md` -- update to write to domain-specific files instead
  - Domain determination: map finding categories to domains (e.g., spec-related findings -> spec-patterns.md, code-related -> code-patterns.md)
  - Pattern ID generation: use next available PAT-{DOMAIN}-XXX number in the target file
  - Pattern entry format must follow FR-009 structure exactly
  - Files to modify: `.github/agents/review-coordinator.agent.md`

### T29-07 - Add pattern retirement logic to Review Coordinator

- **Description**: Update the Review Coordinator to retire patterns that have not been triggered in 10 consecutive reviews. Move retired patterns to the "Retired Patterns" section with status "retired" and a retirement date.
- **Spec refs**: FR-014
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The Review Coordinator SHALL retire patterns not triggered in 10 consecutive reviews (FR-014)
  - [x] Retired patterns SHALL be moved to the "Retired Patterns" section (FR-014.1)
  - [x] Retired patterns SHALL have status set to "retired" (FR-014.2)
  - [x] Retired patterns SHALL include a retirement date (FR-014.3)
  - [x] If review count tracking is unavailable, retirement processing SHALL be deferred (FR-014 error)
- **Test requirements**: BDD
- **Depends on**: T29-06
- **Implementation Guidance**:
  - Add retirement logic after the pattern curation step in the Review Coordinator workflow
  - Track trigger counts: each time a review references a pattern, increment its trigger count; reset after each retirement check
  - State machine: `active -> retired` is a valid transition per companion artifact state-machines.ts
  - Files to modify: `.github/agents/review-coordinator.agent.md`
  - Known pitfall: tracking trigger counts across reviews requires persistent state -- document how the coordinator maintains this (e.g., in pattern file metadata or activity log)

### T29-08 - Add pattern curation commit format

- **Description**: Implement the commit format for pattern file changes in the Review Coordinator. Pattern additions and retirements SHALL be committed following the format specified in FR-015.
- **Spec refs**: FR-015
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Pattern file changes SHALL be committed with explicit file paths: `git add .sdd/reviews/<domain>-patterns.md` (FR-015)
  - [x] Commit message SHALL follow the format: `docs(patterns): add PAT-<DOMAIN>-XXX <pattern title>` (FR-015)
  - [x] If the commit fails, the coordinator SHALL retry once and report the failure if it persists (FR-015 error)
- **Test requirements**: none
- **Depends on**: T29-06
- **Implementation Guidance**:
  - Add commit instructions to the pattern curation section of review-coordinator.agent.md
  - Use explicit `git add` with specific file paths -- never `git add .` or `git add -A`
  - For retirement: `docs(patterns): retire PAT-<DOMAIN>-XXX <pattern title>`
  - Files to modify: `.github/agents/review-coordinator.agent.md`

### T29-09 - Verify schema validation blocks invalid handoffs

- **Description**: End-to-end verification that schema validation correctly blocks a handoff with invalid input. Test: attempt to start the Planner with a spec that has status "Draft" -- schema validation should fail with "Spec must be Validated before planning".
- **Spec refs**: FR-004, FR-005, US-01 Scenario 2
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Given a spec with status "Draft", when the Planner receives the handoff, then schema validation fails with error "Spec must be Validated before planning" (US-01 Scenario 2)
  - [x] Given a validated spec with companion artifacts, when the Planner receives the handoff, then schema validation passes and planning proceeds (US-01 Scenario 1)
  - [x] Given a WP with no implementation files, when the Reviewer receives the handoff, then schema validation fails with error about missing implementation (US-01 Scenario 3)
- **Test requirements**: BDD
- **Depends on**: T29-01, T29-02, T29-03, T29-04
- **Implementation Guidance**:
  - Manual verification: create a test spec with Status: Draft, invoke the Planner, verify it halts with schema validation error
  - Verify the error message matches the expected text from the schema
  - This is an integration verification task -- no new code, just validation of prior tasks

### T29-10 - Verify pattern isolation across domains

- **Description**: Verify that each agent reads only its domain's patterns and ignores others. Verify pattern curation creates entries in the correct domain file.
- **Spec refs**: FR-012, FR-013, US-02 Scenario 2
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Given `code-patterns.md` has 5 active patterns, when the Spec Architect starts, then code patterns are NOT included in prompts (US-02 Scenario 2)
  - [x] Given `spec-patterns.md` has 3 active patterns, when the Spec Architect starts, then all 3 patterns are included in skill prompts (US-02 Scenario 1)
  - [x] Given no patterns file exists for a domain, when the agent runs, then it proceeds without patterns and logs a warning (US-02 Scenario 3)
  - [x] Pattern curation places new patterns in the correct domain-specific file
- **Test requirements**: BDD
- **Depends on**: T29-05, T29-06
- **Implementation Guidance**:
  - Manual verification: inspect each agent's prompt construction to confirm only domain-specific patterns are included
  - Verify Spec Architect does not reference plan-patterns.md, code-patterns.md, or doc-patterns.md
  - Verify Planner does not reference spec-patterns.md, code-patterns.md, or doc-patterns.md
  - Verify Coder does not reference spec-patterns.md, plan-patterns.md, or doc-patterns.md
  - This is an integration verification task

## Implementation Notes

All deliverables are updates to existing agent Markdown files (`.github/agents/*.agent.md`). No executable code, no build system, no test framework. "Testing" means manually invoking agents and verifying behavior matches BDD scenarios.

The schema validation step must come BEFORE all existing workflow steps in each coordinator. This is a structural requirement (FR-005) -- any agent that performs research or skill dispatch before completing schema validation is non-compliant.

The Review Coordinator is the most heavily modified agent in this WP: it gains schema validation (T29-04), formalized pattern consumption (T29-05), automated curation (T29-06), retirement logic (T29-07), and commit format (T29-08).

The legacy `review-patterns.md` reference in review-coordinator.agent.md (lines 364, 435) must be updated to reference domain-specific files instead. This directly relates to WP28's migration.

## Parallel Opportunities

T29-01 through T29-04 can all be worked in parallel (each modifies a different agent file). T29-05 modifies all agent files sequentially. T29-06, T29-07, T29-08 all modify review-coordinator.agent.md and should be sequential. T29-09 and T29-10 are verification steps that run last.

## Risks & Mitigations

- **Risk**: Schema validation adds a blocking step that could break existing handoff flows. **Mitigation**: Validation is instruction-based (Markdown), not executable. If a schema file is missing, the error behavior from FR-001 applies -- report missing file at startup.
- **Risk**: Pattern consumption may already be partially implemented inconsistently. **Mitigation**: T29-05 audits all coordinators for compliance before making changes.
- **Risk**: Review Coordinator modifications are extensive -- regression risk. **Mitigation**: T29-09 and T29-10 are dedicated verification tasks.
- **Risk**: Tracking pattern trigger counts (FR-014) across reviews requires persistent state that may not exist. **Mitigation**: Document a lightweight tracking approach (e.g., count-based annotation in the pattern file itself).

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-06T12:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-06T12:01:00Z - coder - T29-01 - completed - Schema validation added to Spec Architect
- 2026-04-06T12:02:00Z - coder - T29-02 - completed - Schema validation added to Planner
- 2026-04-06T12:03:00Z - coder - T29-03 - completed - Schema validation added to Coder
- 2026-04-06T12:04:00Z - coder - T29-04 - completed - Schema validation added to Review Coordinator
- 2026-04-06T12:05:00Z - coder - T29-05 - completed - Domain-specific pattern isolation formalized
- 2026-04-06T12:06:00Z - coder - T29-06 - completed - Automated pattern curation with domain files
- 2026-04-06T12:07:00Z - coder - T29-07 - completed - Pattern retirement logic added
- 2026-04-06T12:08:00Z - coder - T29-08 - completed - Pattern curation commit format added
- 2026-04-06T12:09:00Z - coder - T29-09 - completed - Schema validation blocking verified by inspection
- 2026-04-06T12:10:00Z - coder - T29-10 - completed - Pattern isolation verified by inspection
- 2026-04-06T12:11:00Z - coder - lane=for_review - All tasks complete, verification passed
- 2026-04-06T14:00:00Z - review-coordinator - lane=done - Verdict: Approved with Findings (2 WARNs)

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-06T14:00:00Z
> **Verdict**: Approved with Findings
> **Skills dispatched**: review-spec (PASS), review-quality (WARN)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All 10 tasks have acceptance criteria checked [x]
- [PASS] Activity Log: Correct lane transitions (planned -> doing -> for_review)
- [WARN] Commit granularity: 5 commits for 10 tasks. T29-05/06/07/08 bundled with T29-01/02/03/04 commits per-file rather than per-task. T29-09/10 are verification-only (no commits expected).
- [PASS] Encoding: No violations found

### Review Feedback

> No FAIL items -- no remediation required.

(No FB-XX items)

### Warnings
- [WARN] PROC-003: Commit granularity -- tasks T29-05 through T29-08 were bundled into T29-01 through T29-04 commits (one commit per agent file rather than one per task). Pragmatic given same-file changes, but deviates from the "one commit per task" policy.
- [WARN] QUAL-004: FR reference ambiguity -- pattern consumption steps reference FR numbers from different spec contexts without disambiguating prefixes (e.g., Coder Step 4 references "FR-004, FR-011, FR-012" where FR-004 is from the Coder spec and FR-011/FR-012 from spec 006). Could cause confusion during maintenance.

### Cross-Correlation Notes
- No cross-correlation findings.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 3 | 1 | 0 |
| review-spec | 15 | 0 | 0 |
| review-quality | 6 | 1 | 0 |
| **Total** | **24** | **2** | **0** |
