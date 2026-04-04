---
lane: done
---

# WP05 - Code Quality Review Skill (review-quality)

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/001-reviewer-v2-skill-based-architecture.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP02 |
| Goal | Create the review-quality skill that evaluates implementation code for readability, complexity, naming, comments, error handling, style consistency, dead code, and duplication |
| Status | Not Started |
| Independent Test | Install the review-quality skill and invoke the coordinator on a WP with known quality issues (dead function, bare except, duplicated logic). Verify: findings file contains FAIL for dead code and bare except, WARN for duplication, with file paths and code evidence |
| Parallelisable | Yes (with WP03, WP04) |
| Prompt | `.sdd/plans/WP05-review-quality.md` |

## Objective

Create `.github/skills/review-quality/SKILL.md` - the code quality review skill. This skill evaluates implementation quality across 8 dimensions: readability, complexity, naming, comments, error handling, style/consistency, dead code, and duplication. It enforces objective quality standards based on codebase conventions (FR-039: no subjective preferences) and uses clear severity rules (FR-038) to distinguish between critical issues (FAIL) and advisory findings (WARN).

## Spec References

- Section 4.2 (FR-025 to FR-029) - Common skill contract
- Section 4.5 (FR-037 to FR-039) - Code quality skill requirements
- Section 7.1 (Skill Findings File format)
- Section 7.5 (Skill File metadata)
- Section 11.2 (BDD scenarios for code quality)

## Tasks

### T05-01 - Create SKILL.md with frontmatter and purpose

- **Description**: Create the file `.github/skills/review-quality/SKILL.md` with YAML frontmatter and purpose statement.
- **Spec refs**: Section 7.5, FR-025
- **Parallel**: No (foundation for all T05 tasks)
- **Acceptance criteria**:
  - [x] File exists at `.github/skills/review-quality/SKILL.md`
  - [x] YAML frontmatter `name` is `review-quality`
  - [x] YAML frontmatter `description` explains: evaluates code quality across 8 dimensions (readability, complexity, naming, comments, error handling, style, dead code, duplication)
  - [x] Purpose section states the skill's role as subagent invoked by coordinator
  - [x] `.gitkeep` file removed from `.github/skills/review-quality/` (replaced by SKILL.md)
- **Test requirements**: none (structural verification)
- **Depends on**: none
- **Implementation Guidance**:
  - Follow the same frontmatter pattern as WP03 (review-spec) and WP04 (review-security):
    ```yaml
    ---
    name: review-quality
    description: "Code quality review skill. Evaluates readability, complexity, naming, comments, error handling, style consistency, dead code, and duplication."
    argument-hint: "Invoked by Review Coordinator - do not call directly"
    ---
    ```

### T05-02 - Write readability, complexity, naming, and comment checklist

- **Description**: Write the first four quality dimensions as checklist items: readability, complexity, naming quality, and comment quality.
- **Spec refs**: FR-037 dimensions 1-4
- **Parallel**: Yes (can be written alongside T05-03)
- **Acceptance criteria**:
  - [x] Readability checks: functions are concise and single-purpose, control flow is straightforward (low nesting depth), code is understandable without extensive comments
  - [x] Complexity checks: flag functions with cyclomatic complexity > 10 (branching, nested conditionals, multiple loops), recommend extraction or simplification
  - [x] Naming quality checks: descriptive intention-revealing names, no single-letter variables outside loop counters, no misleading names, consistent with codebase conventions
  - [x] Comment quality checks: comments explain "why" not "what", no commented-out code, no redundant comments, TODO/FIXME/HACK markers flagged as WARN
  - [x] Each dimension has at least 3 specific, verifiable checklist items
  - [x] Checklist items are phrased as questions the subagent can answer by reading code
- **Test requirements**: BDD - Section 11.2 "High complexity function" scenario
- **Depends on**: T05-01
- **Implementation Guidance**:
  - Complexity threshold: > 10 cyclomatic complexity. The subagent estimates this by counting branching points (if/elif/else, for, while, try/except, ternary, boolean operators in conditions)
  - Naming conventions should be discovered from the codebase (camelCase vs snake_case, etc.), not imposed by the skill
  - Example checklist items:
    - "Are all functions less than 50 lines of meaningful code?"
    - "Does any function have more than 3 levels of nesting?"
    - "Are there any single-letter variable names outside of loop counters (i, j, k)?"
    - "Is there any commented-out code that should be removed?"

### T05-03 - Write error handling, style, dead code, and duplication checklist

- **Description**: Write the remaining four quality dimensions: error handling, style/consistency, dead code, and duplication.
- **Spec refs**: FR-037 dimensions 5-8
- **Parallel**: Yes (can be written alongside T05-02)
- **Acceptance criteria**:
  - [x] Error handling checks: no bare `except` (Python) or empty `catch` blocks, no swallowed exceptions, descriptive error messages, specific exception types, graceful error recovery
  - [x] Style/consistency checks: code follows codebase's established patterns (indentation, bracket style, import ordering, module structure), no inconsistencies introduced by the WP
  - [x] Dead code checks: declared symbols (functions, classes, variables, imports, routes) never referenced anywhere, unreachable code paths (code after return/throw/break)
  - [x] Duplication checks: 3+ lines of identical or near-identical logic in multiple locations flagged
  - [x] Each dimension has at least 3 specific, verifiable checklist items
- **Test requirements**: BDD - Section 11.2 "Dead code detected" and "Bare except handler" scenarios
- **Depends on**: T05-01
- **Implementation Guidance**:
  - Dead code detection: the subagent should use `grep_search` or `vscode_listCodeUsages` to verify if a symbol is referenced anywhere
  - Bare `except` and empty `catch` are language-specific patterns:
    - Python: `except:` without exception type
    - JavaScript: `catch(e) {}` with empty body
    - General: any error/exception handler that silently discards the error
  - Style consistency: the subagent discovers existing codebase patterns first, then checks if the WP's code matches those patterns. It does NOT impose its own preferences.
  - Duplication: "near-identical" means same logic with minor variable name changes. 3+ lines is the threshold per FR-037.

### T05-04 - Write severity guidance

- **Description**: Write clear severity rules mapping each quality dimension to FAIL or WARN levels.
- **Spec refs**: FR-038 (severity rules), FR-039 (no subjective preferences)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] FAIL: dead code (declared but never referenced symbols), unreachable code, bare exception handlers / empty catch blocks
  - [x] WARN: complexity > 10, naming issues, comment issues (TODO/FIXME/HACK), style inconsistencies, duplication, readability concerns
  - [x] WARN items become FAIL only when they "significantly impair maintainability" (FR-038) - include guidance on what "significantly" means: 3+ WARN-level issues of the same type in the same file, or complexity > 20
  - [x] No subjective style preferences enforced (FR-039): only flag deviations from EXISTING codebase patterns, not the skill's or subagent's preferences
  - [x] Explicit instruction: "If you cannot determine the codebase convention for a style question, do not flag it"
- **Test requirements**: BDD - Section 11.2 code quality severity scenarios
- **Depends on**: T05-02, T05-03 (must have all dimensions defined)
- **Implementation Guidance**:
  - The key principle from FR-039: the skill measures consistency with existing patterns, not adherence to any external style guide
  - "Significantly impair maintainability" threshold guidance:
    - A single TODO comment = WARN
    - 5+ TODO comments indicating deferred work = discussion-worthy (still WARN, but noted as pattern)
    - A single dead function = FAIL (dead code is always FAIL per FR-038)
    - A function with complexity 12 = WARN; complexity 25 = FAIL (significantly impairs readability)

### T05-05 - Write output format instructions

- **Description**: Write the output format instructions matching Section 7.1, including the read-only constraint and findings template.
- **Spec refs**: FR-027 (findings format), FR-028 (read-only), FR-029 (N/A handling)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Output format matches Section 7.1: YAML frontmatter with skill, wp, spec, reviewed_at, status, finding_counts, files_reviewed
  - [x] Finding prefix is `QUAL-` (e.g., `QUAL-001`, `QUAL-002`)
  - [x] Each FAIL/WARN finding includes: checklist item reference, requirement ref, file path with line range, description, expected behavior, evidence
  - [x] Each PASS finding includes: checklist item, file path, description
  - [x] N/A items include justification
  - [x] Finding IDs are sequential, `finding_counts` matches actual findings
  - [x] `files_reviewed` lists all files evaluated
  - [x] Read-only constraint: "Do NOT modify any source code, WP file, or spec file" (FR-028)
  - [x] Complete example findings file included in skill instructions
- **Test requirements**: BDD - format verification
- **Depends on**: T05-04 (must have severity rules for the example)
- **Implementation Guidance**:
  - Follow the same output format pattern as WP03 (review-spec) with prefix `QUAL-`
  - Example finding:
    ```markdown
    ### QUAL-001 [FAIL]
    - **Checklist item**: Dead Code - Unused function
    - **Requirement**: FR-037 dimension 7
    - **File**: src/utils/helpers.py#L120-L135
    - **Description**: Function `process_legacy()` is defined but never called anywhere.
    - **Expected**: Remove unused function or document why it is retained.
    - **Evidence**: `grep_search` for `process_legacy` returns only the definition, no call sites.
    ```
  - Include examples for WARN (complexity) and N/A (e.g., "No database code in this WP, duplication check for SQL patterns N/A")

## Implementation Notes

- This is a SINGLE file: `.github/skills/review-quality/SKILL.md`
- File structure: Purpose -> Quality Checklist (8 dimensions) -> Severity Rules -> Output Format
- Target size: 150-220 lines. The quality checklist is shorter than security (8 dimensions vs 14 categories).
- Key principle: FR-039 prohibits subjective preferences. The skill measures consistency with existing codebase patterns, not adherence to any external style guide. This must be emphasized prominently in the skill file.
- The complexity threshold (> 10) is a guideline, not a hard rule. The subagent should use judgment for edge cases (a function at 11 with clear logic is different from one at 11 with confusing control flow).

## Parallel Opportunities

- T05-02 and T05-03 can run concurrently (two halves of the quality dimensions)
- All other tasks are sequential

## Risks & Mitigations

- **Risk**: Subagent enforces its own style preferences instead of codebase conventions
  - Mitigation: FR-039 constraint is included in the skill with guidance: "Discover existing patterns first. If you cannot determine the convention, do not flag." Include examples of what NOT to do.
- **Risk**: Dead code detection produces false positives for exported symbols or framework hooks
  - Mitigation: Include guidance that framework hooks (lifecycle methods, signal handlers, route handlers) are not "dead code" even if not explicitly called. Only flag symbols with zero references AND no framework registration.
- **Risk**: Complexity estimation is inaccurate without tooling
  - Mitigation: The subagent counts branching points manually. This is approximate but sufficient for flagging obvious complexity. The threshold (> 10) has margin for estimation error.

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-04T23:45:00Z
> **Verdict**: Changes Required
> **Skills dispatched**: review-spec (FAIL), review-security (WARN), review-quality (WARN), review-tests (PASS), review-architecture (PASS), review-performance (PASS), review-docs (WARN), review-deps (PASS)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All 5 tasks have acceptance criteria checked off
- [PASS] Activity Log: Consistent lane transitions planned -> doing -> for_review
- [WARN] Commit granularity: Single commit (a16b597) for entire WP rather than per-task commits
- [PASS] Encoding: No prohibited Unicode characters found

### Review Feedback

> Implementers: address every FB-XX item before returning for re-review.

- [x] **FB-01**: [spec-adherence] FR-027 Partial - PASS findings output format omits `Requirement` field. Section 7.1 requires this field for all severities including PASS. The peer skill review-spec correctly includes it.
  File: .github/skills/review-quality/SKILL.md (output format rules for PASS findings).
  Expected: Change PASS finding rule to include `Requirement` between `Checklist item` and `File`.
  Source skill: review-spec (SPEC-003)

### Warnings
- [WARN] PROC-003: Single commit for entire WP.
- [WARN] SEC-005: Missing explicit NFR-004 constraint (no code execution). Defense-in-depth recommendation -- review-security includes this, review-quality does not.
- [WARN] SEC-006: Missing explicit NFR-005 constraint (no secret reproduction in findings). Defense-in-depth recommendation.
- [WARN] QUAL-008: Missing PASS example finding in output format template.
- [WARN] DOC-004: Architecture.md lists only 4 of 8 quality dimensions in skills table description.
- [WARN] DOC-016: 3 of 6 standard doc files missing (pre-existing N/A-domain condition, downgraded from subagent FAIL per prior review precedent WP03/WP04).

### Cross-Correlation Notes
- DOC-016 downgraded from FAIL to WARN: Same pre-existing condition as WP03 DOC-020 (WARN) and WP04 DOC-020 (WARN). The 3 missing files cover domains N/A for this project (no APIs, no config, no deployment). Maintaining severity consistency across WPs.
- No duplicates detected between skills.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 3 | 1 | 0 |
| review-spec | 14 | 0 | 1 |
| review-security | 5 | 2 | 0 |
| review-quality | 6 | 1 | 0 |
| review-tests | 1 | 0 | 0 |
| review-architecture | 14 | 0 | 0 |
| review-performance | 0 | 0 | 0 |
| review-docs | 17 | 2 | 0 |
| review-deps | 0 | 0 | 0 |
| **Total** | **60** | **6** | **1** |

## Activity Log

- 2026-04-04T11:30:00Z - planner - lane=planned - Work package created
- 2026-04-04T15:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-04T15:15:00Z - coder - lane=for_review - All tasks complete, submitted for review
- 2026-04-04T23:45:00Z - review-coordinator - lane=to_do - Verdict: Changes Required (1 FAIL) -- awaiting remediation
- 2026-04-04T23:50:00Z - coder - lane=doing - Addressing reviewer feedback (FB-01)
- 2026-04-04T23:52:00Z - coder - lane=for_review - FB-01 resolved, submitted for re-review

### Round 2 Re-Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-04T23:55:00Z
> **Verdict**: Approved with Findings
> **Skills re-dispatched**: review-spec (PASS - was FAIL)
> **Skills unchanged**: review-security, review-quality, review-tests, review-architecture, review-performance, review-docs, review-deps
> **Review round**: 2

**FB-01 Resolution**: Verified. `Requirement` field added to PASS finding rules at line 163. SPEC-003 upgraded FAIL -> PASS.

**Remaining Warnings** (informational, not blocking):
- [WARN] PROC-003: Single commit for initial WP implementation
- [WARN] SEC-005: Missing explicit NFR-004 constraint (defense-in-depth)
- [WARN] SEC-006: Missing explicit NFR-005 constraint (defense-in-depth)
- [WARN] QUAL-008: Missing PASS example finding in output format
- [WARN] DOC-004: Architecture.md lists only 4 of 8 quality dimensions
- [WARN] DOC-016: 3 N/A-domain doc files missing (pre-existing)

### Round 2 Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 3 | 1 | 0 |
| review-spec (R2) | 15 | 0 | 0 |
| review-security (R1) | 5 | 2 | 0 |
| review-quality (R1) | 6 | 1 | 0 |
| review-tests (R1) | 1 | 0 | 0 |
| review-architecture (R1) | 14 | 0 | 0 |
| review-performance (R1) | 0 | 0 | 0 |
| review-docs (R1) | 17 | 2 | 0 |
| review-deps (R1) | 0 | 0 | 0 |
| **Total** | **61** | **6** | **0** |

- 2026-04-04T23:55:00Z - review-coordinator - lane=done - Verdict: Approved with Findings (0 FAIL, 6 WARN)
