---
lane: done
---

# WP16 - Phase 1: Decomposition + Acceptance Skills

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/003-planner-v2.spec.md` |
| Priority | P1 |
| Lane | doing |
| Depends on | WP14, WP15 |
| Goal | Implement the two Phase 1 planning skills that decompose a spec into work packages with tasks, acceptance criteria, implementation guidance, and traceability |
| Status | Complete |
| Independent Test | Dispatch plan-decomposition and plan-acceptance against a validated spec. Verify: WP files exist with 5-12 tasks each, every task has 3+ acceptance criteria with SHALL statements, implementation guidance with doc links, every FR assigned to exactly one task, README has WP index and MVP scope |
| Parallelisable | No |
| Prompt | `.sdd/plans/WP16-phase1-decomposition-acceptance.md` |

## Objective

Implement the two Phase 1 planning skills: `plan-decomposition` (WP identification, task breakdown, sequencing, dependencies) and `plan-acceptance` (acceptance criteria extraction, implementation guidance, spec traceability). These skills produce the plan accumulator (WP files and README) that Phase 2 contract skills consume.

## Spec References

FR-028 through FR-036, Section 4.3, Section 4.4, Section 7.1 (Plan Accumulator data model)

## Tasks

### T16-01 - Implement plan-decomposition SKILL.md structure

- **Description**: Replace the stub `plan-decomposition/SKILL.md` with the full skill implementation. Define the skill's purpose, input contract (referencing PLAN-SKILL-CONTRACT.md), execution sequence, and output format.
- **Spec refs**: FR-028, FR-023, FR-024
- **Parallel**: No
- **Acceptance criteria**:
  - [x] SKILL.md follows the common plan-skill contract (9 inputs from FR-023, 4-step execution from FR-024)
  - [x] Skill reads its own SKILL.md, then reads plan state, then reads spec + artifacts (FR-024)
  - [x] Skill references `.github/skills/PLAN-SKILL-CONTRACT.md` for the common contract
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Pattern: Mirror the structure of `.github/skills/spec-requirements/SKILL.md`
  - The skill file is instructions for an LLM subagent -- it describes WHAT to produce and HOW, not executable code

### T16-02 - Implement WP identification and sequencing logic

- **Description**: Write the core decomposition instructions: analyze spec FRs, user stories, and architecture; identify logical work packages following the sequencing logic (Foundation -> Core domain -> Integrations -> User-facing -> Quality -> Delivery); decompose each WP into 5-12 tasks.
- **Spec refs**: FR-028 (items 1-5), FR-030
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL analyze the spec's functional requirements, user stories, and architecture (FR-028.1)
  - [x] The skill SHALL identify logical WPs following: Foundation -> Core domain -> Integrations -> User-facing -> Quality -> Delivery (FR-028.2)
  - [x] The skill SHALL decompose each WP into 5-12 atomic tasks (FR-028.3, FR-030)
  - [x] The skill SHALL define inter-task and inter-WP dependencies by ID (FR-028.4)
  - [x] The skill SHALL assign priorities: P0 (foundation), P1 (MVP user story), P2+ (incremental) (FR-028.5)
- **Test requirements**: BDD (US-01 Scenario 1: all FRs assigned, Scenario 2: foundation WP, Scenario 3: parallel WPs)
- **Depends on**: T16-01
- **Implementation Guidance**:
  - Sequencing logic from FR-028.2 is a guideline, not rigid -- adjust based on spec dependencies
  - FR-030: fewer than 5 tasks suggests the WP is too granular; more than 12 suggests splitting

### T16-03 - Implement WP file template and metadata table

- **Description**: Write the WP file output template including the metadata table header (FR-031), task skeleton format (FR-029), and skeleton README with WP index (FR-028.7).
- **Spec refs**: FR-028 (items 6-7), FR-029, FR-031, FR-032
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Every WP file SHALL include the metadata table from FR-031 with fields: Spec, Priority, Lane, Depends on, Goal, Status, Independent Test (FR-031)
  - [x] Each task SHALL include: unique T<NN>-XX identifier, Description, Spec refs, Parallel flag, placeholder for acceptance criteria, placeholder for implementation guidance, Dependencies (FR-029)
  - [x] The skill SHALL write a skeleton README with WP index and dependency graph (FR-028.7)
  - [x] The first task of the foundation WP SHALL be virtual environment setup for languages with package isolation (FR-032)
- **Test requirements**: BDD (US-01 Scenario 2: foundation WP with venv setup as T01-01)
- **Depends on**: T16-02
- **Implementation Guidance**:
  - FR-029 task fields are placeholders -- plan-acceptance fills acceptance criteria and implementation guidance
  - FR-032: check the spec's tech stack section to determine if package isolation is needed
  - The metadata table format matches the existing WP template used by WP01-WP13

### T16-04 - Implement plan-acceptance SKILL.md structure

- **Description**: Replace the stub `plan-acceptance/SKILL.md` with the full skill implementation. Define the skill's purpose, input contract, execution sequence, and output format. This skill reads the WP files produced by plan-decomposition and populates acceptance criteria and implementation guidance.
- **Spec refs**: FR-033, FR-023, FR-024
- **Parallel**: No
- **Acceptance criteria**:
  - [x] SKILL.md follows the common plan-skill contract (9 inputs from FR-023)
  - [x] Skill reads existing WP files produced by plan-decomposition before writing (FR-016, FR-024)
  - [x] Skill references `.github/skills/PLAN-SKILL-CONTRACT.md` for the common contract
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - This skill runs AFTER plan-decomposition in Phase 1 sequence
  - It must read WP files that already have task skeletons with placeholders

### T16-05 - Implement acceptance criteria extraction

- **Description**: Write the instructions for copying exact SHALL statements from spec FRs as acceptance criteria (min 3 per task), copying Given/When/Then scenarios from user stories, and specifying test requirements per task.
- **Spec refs**: FR-033 (items 1-4), FR-035
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL copy exact SHALL statements from the spec's FRs as acceptance criteria, at least 3 per task (FR-033.1)
  - [x] The skill SHALL copy acceptance scenarios from user stories as Given/When/Then (FR-033.2)
  - [x] The skill SHALL add implementation guidance with official doc links, recommended patterns, known pitfalls, error codes, and validation rules (FR-033.3)
  - [x] The skill SHALL specify test requirements per task: unit / integration / BDD / E2E / none (FR-033.4)
  - [x] The skill SHALL include BDD/TDD requirements in every task with test requirements: tests derive from spec acceptance scenarios, not from implementation (FR-035)
- **Test requirements**: BDD (US-01 Scenario 1)
- **Depends on**: T16-04
- **Implementation Guidance**:
  - FR-033.1: "at least 3 per task" is a hard minimum -- if a task has fewer than 3 applicable SHALL statements, the skill must derive additional criteria from related FRs
  - FR-035: BDD/TDD emphasis means test tasks reference spec scenarios, not implementation details
  - Known pitfalls: Acceptance criteria must be copy-pasted from spec FRs, not paraphrased

### T16-06 - Implement FR traceability verification

- **Description**: Write the traceability verification logic: every FR in the spec's traceability matrix (Section 16) must be assigned to exactly one task. Unassigned FRs get assigned; duplicate assignments get resolved.
- **Spec refs**: FR-034
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL verify every FR in Section 16 is assigned to exactly one task (FR-034)
  - [x] If an FR is unassigned, the skill SHALL assign it to the most relevant task (FR-034)
  - [x] If an FR is assigned to multiple tasks, the skill SHALL resolve the duplication by choosing the primary task (FR-034)
  - [x] After verification, zero orphan FRs SHALL remain
- **Test requirements**: BDD (US-01 Scenario 1: all FRs assigned, no orphans)
- **Depends on**: T16-05
- **Implementation Guidance**:
  - Cross-reference Section 16 traceability matrix against task spec refs across all WP files
  - Resolution strategy: if an FR maps to multiple tasks, keep it on the task most directly responsible for implementing it

### T16-07 - Implement README updates

- **Description**: Write the README update logic: populate the plan index with complete WP table (status, priority, dependencies), MVP scope, and dependency graph.
- **Spec refs**: FR-036
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL update README with a complete WP index table with status, priority, dependencies (FR-036.1)
  - [x] The skill SHALL identify and mark MVP scope -- which WPs constitute the minimum releasable increment (FR-036.2)
  - [x] The skill SHALL produce a dependency graph showing WP sequencing (FR-036.3)
- **Test requirements**: none
- **Depends on**: T16-06
- **Implementation Guidance**:
  - MVP scope: P0 + P1 WPs are typically MVP; P2+ are post-MVP
  - Dependency graph can be mermaid or prose; must be acyclic

### T16-08 - Verify encoding compliance

- **Description**: Run an automated check on both skill files to verify no prohibited Unicode characters. Fix any violations.
- **Spec refs**: Section 9.2
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Zero prohibited Unicode characters in plan-decomposition/SKILL.md
  - [x] Zero prohibited Unicode characters in plan-acceptance/SKILL.md
  - [x] All hyphens are ASCII `-` (U+002D), all quotes are straight
- **Test requirements**: unit (encoding validation)
- **Depends on**: T16-07
- **Implementation Guidance**:
  - Reuse the Python encoding check from WP14

## Implementation Notes

- Phase 1 skills produce the plan accumulator: WP markdown files and README
- plan-decomposition creates the structure; plan-acceptance fills in the quality details
- The two skills execute sequentially -- plan-acceptance reads plan-decomposition's output
- These skills produce NO contract files (that is Phase 2's responsibility)

## Parallel Opportunities

- T16-01 and T16-04 (skill structure setup) can be worked in parallel since they are independent skills
- All other tasks within each skill are sequential

## Risks & Mitigations

- **Risk**: plan-acceptance may produce inconsistent acceptance criteria if it cannot find exact SHALL statements for every task. **Mitigation**: FR-033 allows derived criteria when spec FRs do not map 1:1 to tasks.
- **Risk**: Traceability verification (FR-034) may discover FRs that do not fit any existing task. **Mitigation**: The skill SHALL assign orphan FRs to the most relevant task, potentially adding them to Implementation Guidance rather than acceptance criteria.

## Activity Log

- 2026-04-05T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-05T12:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-05T12:30:00Z - coder - lane=for_review - All tasks complete, submitted for review
- 2026-04-05T11:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-05T18:00:00Z - review-coordinator - lane=to_do - Verdict: Changes Required (1 FAIL) -- awaiting remediation
- 2026-04-05T19:30:00Z - coder - lane=for_review - Remediated FB-01 (FR-033.3 guidance template: added error codes and validation rules fields)
- 2026-04-05T21:00:00Z - review-coordinator - lane=done - Verdict: Approved

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-05T21:00:00Z
> **Verdict**: Approved
> **Skills dispatched**: review-spec (PASS)
> **Review round**: 2

### Process Compliance
- [PASS] Spec Compliance Checklist: All acceptance criteria checked
- [PASS] Activity Log: Consistent transitions (remediation entry present)
- [PASS] Commit granularity: Remediation commit (33f8870) targets specific fix
- [PASS] Encoding: No violations found

### Review Feedback

> No FAIL findings. Previous FB-01 has been resolved.

### Warnings

(none)

### Cross-Correlation Notes
- No cross-correlation findings.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 4 | 0 | 0 |
| review-spec | 12 | 0 | 0 |
| **Total** | **16** | **0** | **0** |
