---
lane: planned
---

# WP03 - Spec Adherence Review Skill (review-spec)

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/001-reviewer-v2-skill-based-architecture.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP02 |
| Goal | Create the review-spec skill that evaluates whether implementation code faithfully implements every functional requirement, success criterion, and acceptance scenario from the specification |
| Status | Not Started |
| Independent Test | Install the review-spec skill and invoke the coordinator on a WP with a known spec deviation (e.g., missing FR endpoint, wrong error code). Verify: the skill produces a findings file with FAIL findings citing the deviating FRs, and the coordinator includes them in the FB-XX list |
| Parallelisable | Yes (with WP04, WP05) |
| Prompt | `.sdd/plans/WP03-review-spec.md` |

## Objective

Create `.github/skills/review-spec/SKILL.md` - the spec adherence review skill. This is the first skill implemented and establishes the reference pattern for all subsequent skills. It verifies that every functional requirement (FR-XXX) referenced by a work package is faithfully implemented: SHALL obligations are satisfied, preconditions enforced, postconditions produced, error paths handled, edge cases covered, data models match, API contracts match, and stubs are detected. This is the spec's P1 priority and the most critical review dimension (spec drift is the primary quality risk).

## Spec References

- Section 4.2 (FR-025 to FR-029) - Common skill contract
- Section 4.3 (FR-030 to FR-033) - Spec adherence specific requirements
- Section 7.1 (Skill Findings File format)
- Section 7.5 (Skill File metadata)
- Section 8.3 (Skill Subagent Prompt Interface)
- Section 11.2 (BDD scenarios for spec adherence)

## Tasks

### T03-01 - Create SKILL.md with frontmatter and purpose

- **Description**: Create the file `.github/skills/review-spec/SKILL.md` with YAML frontmatter (name, description, argument-hint) and a purpose statement explaining what this skill does and when it runs.
- **Spec refs**: Section 7.5 (Skill File metadata), FR-025 (input contract)
- **Parallel**: No (foundation for all T03 tasks)
- **Acceptance criteria**:
  - [ ] File exists at `.github/skills/review-spec/SKILL.md`
  - [ ] YAML frontmatter `name` is `review-spec`
  - [ ] YAML frontmatter `description` explains the skill's purpose: evaluating spec adherence by comparing implementation against functional requirements
  - [ ] Purpose section states: this skill is invoked by the Review Coordinator as a subagent, receives WP and spec paths, discovers and reads implementation code, evaluates each FR, and writes structured findings
  - [ ] Purpose section references the common input contract (FR-025): reads SKILL.md, reads spec, discovers code, evaluates checklist, writes findings
  - [ ] `.gitkeep` file removed from `.github/skills/review-spec/` (replaced by SKILL.md)
- **Test requirements**: none (structural verification)
- **Depends on**: none
- **Implementation Guidance**:
  - Reference existing skill pattern from `.github/skills/semantic-commit/SKILL.md` for YAML frontmatter format:
    ```yaml
    ---
    name: review-spec
    description: "Spec adherence review skill. Evaluates implementation against functional requirements, success criteria, and acceptance scenarios from the specification."
    argument-hint: "Invoked by Review Coordinator - do not call directly"
    ---
    ```
  - The skill file body is natural language instructions that a subagent reads and follows
  - Keep the purpose section concise (5-10 lines) - the details come in subsequent tasks
  - These skills are called by the coordinator per Section 8.3 prompt template, not directly by users

### T03-02 - Write FR classification and adherence checklist

- **Description**: Write the core checklist that the skill follows to evaluate each functional requirement. This includes the FR classification system (Compliant/Partial/Deviating/Missing) and the detailed verification checklist for each FR.
- **Spec refs**: FR-030 (FR classification), FR-031 (detailed verification checklist)
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] Checklist instructs: for each FR referenced by the WP's `Spec References` section, classify adherence as Compliant, Partial, Deviating, or Missing
  - [ ] Classification definitions match spec exactly: Compliant = fully implemented as specified; Partial = some aspects missing; Deviating = implemented but behaves differently; Missing = not implemented at all
  - [ ] Partial, Deviating, or Missing classification produces a FAIL finding
  - [ ] Detailed verification checklist per FR includes all 8 items from FR-031:
    - SHALL/SHALL NOT obligation satisfied exactly
    - Preconditions enforced in code
    - Postconditions produced by code
    - Error paths handled as specified
    - Edge cases from acceptance scenarios covered
    - Data model fields/types/validation match Section 7
    - API request/response schemas match Section 8
    - Error codes match spec error taxonomy
  - [ ] Each checklist item that fails produces evidence (code snippet, expected vs actual behavior)
- **Test requirements**: BDD - Section 11.2 "FR fully implemented", "FR partially implemented" scenarios
- **Depends on**: T03-01
- **Implementation Guidance**:
  - The skill reads the spec file to extract FRs, then reads the WP file to determine which FRs are in scope
  - For each in-scope FR: locate the implementation code, evaluate against all 8 checklist items
  - The classification determines the finding severity: Compliant -> PASS, everything else -> FAIL
  - Include specific instruction to check error paths - this is the most commonly missed aspect per the current reviewer's experience
  - The skill should find relevant code using `grep_search` and `semantic_search` tools

### T03-03 - Write stub detection rules

- **Description**: Write instructions for detecting stub implementations that should be classified as Missing, not Partial.
- **Spec refs**: FR-032 (stub detection)
- **Parallel**: Yes (can be written alongside T03-04)
- **Acceptance criteria**:
  - [ ] Skill checks for stub patterns: `pass` (alone in function body), `raise NotImplementedError`, `...` (ellipsis), `# TODO`, empty function bodies, any placeholder that makes a test vacuously pass
  - [ ] Stubs are classified as Missing (not Partial) per FR-032
  - [ ] Missing classification produces a FAIL finding with evidence showing the stub code
  - [ ] The skill explicitly lists all stub patterns to check (no reliance on "judgment")
- **Test requirements**: BDD - Section 11.2 "Stub detected as Missing" scenario
- **Depends on**: T03-01
- **Implementation Guidance**:
  - Stub patterns to scan for (language-agnostic list):
    - Python: `pass`, `raise NotImplementedError`, `...`, `# TODO`, `# FIXME`, `# HACK`
    - JavaScript/TypeScript: `throw new Error("Not implemented")`, empty function bodies `{}`, `// TODO`
    - General: any function body shorter than 3 meaningful lines (excluding comments and whitespace)
  - Use `grep_search` to scan for these patterns in implementation files
  - The distinction between Partial and Missing matters for the Coder: Partial means some logic exists but is incomplete, Missing means no real implementation exists

### T03-04 - Write success criteria verification rules

- **Description**: Write instructions for verifying success criteria (SC-XXX) referenced by the work package.
- **Spec refs**: FR-033 (success criteria verification)
- **Parallel**: Yes (can be written alongside T03-03)
- **Acceptance criteria**:
  - [ ] Skill checks each SC-XXX referenced by the WP for evidence: passing test, observable behavior, or measurable metric
  - [ ] Evidence is verified as genuine (the claimed test or metric actually exists - not fabricated)
  - [ ] SC-XXX that cannot be verified at this stage are documented with a reason (not marked as FAIL)
  - [ ] Missing evidence for a verifiable SC-XXX produces a FAIL finding
  - [ ] Fabricated evidence (claimed test does not exist, claimed metric is not measured) produces a FAIL finding
- **Test requirements**: BDD - Section 11.2 spec adherence scenarios
- **Depends on**: T03-01
- **Implementation Guidance**:
  - Success criteria are in Section 2 of the spec (SC-001 through SC-007 for this project)
  - The WP file's Spec References section lists which SC criteria are relevant
  - "Evidence" means: a test file that exercises the criterion, an observable behavior that can be demonstrated, or a metric that can be measured
  - "Fabricated" means: the WP claims a test exists but the test file does not exist, or the test is vacuous (asserts True)
  - SC criteria that depend on runtime behavior not yet available (e.g., "Add a new skill and verify dispatch") can be marked as "deferred verification" with explanation

### T03-05 - Write severity guidance and N/A handling

- **Description**: Write clear severity rules that tell the subagent exactly which findings should be FAIL, WARN, PASS, or N/A. Include N/A handling per FR-029.
- **Spec refs**: FR-029 (N/A with justification), FR-030 (classification -> severity mapping)
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] FAIL: any FR classified as Partial, Deviating, or Missing; any SC-XXX with missing or fabricated evidence; any stub implementation
  - [ ] PASS: FR classified as Compliant with all 8 checklist items verified
  - [ ] WARN: (none for spec adherence - spec compliance is binary, not gradual)
  - [ ] N/A: FR or checklist item not applicable to this WP's codebase (with justification string, e.g., "No API endpoints in this WP")
  - [ ] Every N/A finding MUST include a justification field explaining why the item does not apply
  - [ ] The skill explicitly states: "Do not produce findings for checklist items that are not applicable"
- **Test requirements**: BDD - Section 11.2 non-applicable category scenarios
- **Depends on**: T03-02 (severity depends on classification system)
- **Implementation Guidance**:
  - Spec adherence is binary: either the FR is implemented correctly or it is not. There is no WARN level for this skill.
  - N/A examples: "FR-045 (performance checks) is not relevant to this skill's evaluation", "No database access in this WP so data model field checks are N/A"
  - The N/A justification prevents false negatives where the skill skips checks without explanation

### T03-06 - Write output format instructions

- **Description**: Write the output format instructions that tell the subagent exactly how to write the findings file, including YAML frontmatter structure and per-finding markdown format from Section 7.1.
- **Spec refs**: FR-027 (findings format), FR-028 (read-only constraint), Section 7.1 (Skill Findings File format)
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] Output format matches Section 7.1 exactly: YAML frontmatter with skill, wp, spec, reviewed_at, status, finding_counts, files_reviewed fields
  - [ ] Each finding uses the heading format: `### <ID> [SEVERITY]`
  - [ ] Finding prefix is `SPEC-` (e.g., `SPEC-001`, `SPEC-002`)
  - [ ] Each FAIL/WARN finding includes: checklist item, requirement ref, file path with line range, description, expected behavior, evidence (code snippet)
  - [ ] Each PASS finding includes: checklist item, requirement ref, file path, description
  - [ ] Each N/A finding includes: checklist item, justification
  - [ ] Finding IDs are sequential (no gaps)
  - [ ] `finding_counts` in frontmatter accurately reflects actual findings in the file
  - [ ] `files_reviewed` lists all files the skill read and evaluated
  - [ ] Explicit instruction: "Do NOT modify any source code, WP file, or spec file" (FR-028)
- **Test requirements**: BDD - format verification
- **Depends on**: T03-02, T03-05 (must know what findings to produce and their severities)
- **Implementation Guidance**:
  - Include a complete example findings file in the skill instructions so the subagent has a concrete reference
  - Example finding entry:
    ```markdown
    ### SPEC-001 [FAIL]
    - **Checklist item**: FR classification - SHALL obligation
    - **Requirement**: FR-012
    - **File**: src/api/users.py#L45-L52
    - **Description**: POST /users endpoint does not return created user object.
    - **Expected**: POST /users returns 201 with full user schema per FR-012.
    - **Evidence**:
      ```python
      return {"status": "created"}  # Missing user object fields
      ```
    ```
  - The `files_reviewed` field is critical for re-review scoping (FR-021) - the coordinator uses it to determine if a PASSing skill needs re-dispatch when files change
  - Finding ID validation: SPEC-001, SPEC-002, ... with no gaps in sequence

## Implementation Notes

- This is a SINGLE file: `.github/skills/review-spec/SKILL.md`
- All tasks contribute sections to this one file
- The file should be organized: Purpose -> Input Instructions -> Checklist -> Stub Detection -> SC Verification -> Severity Rules -> Output Format
- Target size: 150-250 lines. Must stay under 300 lines (constraint C-002)
- This is the FIRST skill implemented and serves as the reference pattern for WP04 (review-security) and WP05 (review-quality). Ensure the structure is clean and reusable.
- The skill is language-agnostic (A-005) - checklist items apply to any programming language

## Parallel Opportunities

- T03-03 and T03-04 can run concurrently (stub detection and SC verification are independent sections)
- All other tasks are sequential

## Risks & Mitigations

- **Risk**: Skill file exceeds 300-line context limit (C-002)
  - Mitigation: Keep instructions concise. Use bullet lists for checklists. Include one example, not many. Target 200 lines.
- **Risk**: Subagent does not follow output format precisely
  - Mitigation: Include a complete example findings file in the skill. The example serves as a template the subagent can copy.
- **Risk**: FR classification is ambiguous for edge cases (is a missing error handler Partial or Deviating?)
  - Mitigation: Provide clear classification decision tree: Missing = no code at all, Partial = some code but incomplete, Deviating = code exists but behaves differently from spec.

## Activity Log

- 2026-04-04T11:20:00Z - planner - lane=planned - Work package created
