# Reviewer V2 - Skill-Based Review Architecture -- Specification

> **Source brief**: `.sdd/ideas/001-reviewer-v2-skill-based-architecture.md`
> **Feature branch**: `002-reviewer-v2`
> **Status**: Draft
> **Version**: 1.0

---

## 1. Overview

Replace the monolithic "5. Reviewer" agent (350+ lines, 13 review dimensions, embedded pipeline orchestration) with a lightweight Review Coordinator agent that dynamically discovers and dispatches specialized review skills via subagents. Each skill focuses on a single review domain with expert-level depth, runs in its own context window, and writes structured findings to persistent files. The coordinator aggregates findings, cross-correlates across dimensions, produces a verdict with actionable feedback items, manages WP lifecycle (lane, review_status, activity log), curates a review-patterns file that teaches the Coder agent to avoid recurring mistakes, and commits the result. Pipeline orchestration (auto-continuation, next-WP routing) is removed from the reviewer and left to the Orchestrator agent.

---

## 2. Goals & Success Criteria

- **SC-001**: Each review skill runs in a fresh context window, enabling deeper analysis per dimension than the current monolithic pass. Verified by: each subagent loads only its SKILL.md plus code -- no other skill checklists compete for context.
- **SC-002**: Security reviews cover all 14 OWASP Secure Coding Practices categories (where applicable to the codebase), compared to the current 3-bullet summary. Verified by: the review-security skill file contains checklist items for all 14 categories.
- **SC-003**: Adding or modifying a review dimension requires editing exactly one focused skill file (under 300 lines), not a 350-line monolith. Verified by: each skill file is self-contained with no cross-skill dependencies.
- **SC-004**: The Coder agent can read a structured patterns checklist before implementation and avoid repeating previously-caught mistakes. Verified by: `.sdd/reviews/review-patterns.md` exists and is referenced by the Coder.
- **SC-005**: Review findings are preserved per-WP per-skill for audit trail. Verified by: `.sdd/reviews/<WP-id>/` contains one findings file per dispatched skill after every review.
- **SC-006**: Cross-correlation detects when the same issue is flagged by multiple skills and produces a single composite finding. Verified by: coordinator merges duplicate findings in the summary report.
- **SC-007**: The coordinator dynamically discovers available review skills at runtime. Verified by: adding a new `review-*` skill directory causes it to be dispatched in the next review without editing the coordinator.

---

## 3. Users & Roles

- **Coder Agent (primary consumer)**: Receives review findings as structured FB-XX items in the WP file. Each FB-XX item is independently actionable -- cite file, line, requirement, and expected fix. The Coder also reads `.sdd/reviews/review-patterns.md` before implementing a new WP to avoid repeating past mistakes.

- **Human Developers (secondary consumer)**: Read review reports in WP files for summary verdict and FB-XX checklist. Read detailed findings in `.sdd/reviews/<WP-id>/` for audit purposes, root cause analysis, or dispute resolution. Findings SHALL be precise, cited, and non-ambiguous.

- **Orchestrator Agent (tertiary consumer)**: Reads the WP `lane:` frontmatter value to determine pipeline state. A `lane: done` means the review passed; `lane: to_do` means changes are required. The Orchestrator routes accordingly. The coordinator does NOT invoke other agents for pipeline continuation -- that is the Orchestrator's responsibility.

- **Spec Architect / Planner Agents (occasional consumer)**: Receive handoff when the coordinator identifies spec gaps or plan-level issues during review. These are triggered via handoff buttons, not automatic invocation.

---

## 4. Functional Requirements

### 4.1 Review Coordinator Agent

#### 4.1.1 Scope Selection

- **FR-001**: The coordinator SHALL accept a WP identifier (e.g., `WP01`) as input, or scan `.sdd/plans/WP*.md` for work packages with `lane: for_review` if no identifier is provided.
  - Precondition: At least one WP file exists with `lane: for_review`.
  - Error: If no WP is ready for review, the coordinator SHALL inform the user and halt.
  - Error: If the specified WP does not exist, the coordinator SHALL list available WPs and ask the user to select one.

- **FR-002**: The coordinator SHALL load the full artifact chain for the selected WP before dispatching any skill:
  1. Work package plan (`.sdd/plans/WP<NN>-*.md`)
  2. Specification (`.sdd/specs/<name>.spec.md`, resolved from the WP's `Spec` field)
  3. Ideation brief (`.sdd/ideas/<name>.md`, resolved from the spec's `Source brief` field)
  4. Plan index (`.sdd/plans/README.md`)
  - Error: If any artifact in the chain is missing or unreadable, the coordinator SHALL halt and report which artifact is missing.

#### 4.1.2 Dynamic Skill Discovery

- **FR-003**: The coordinator SHALL discover available review skills by scanning for directories matching the glob pattern `.github/skills/review-*/SKILL.md` at the start of each review.
  - Postcondition: A sorted list of discovered skill names (e.g., `review-spec`, `review-security`, `review-quality`) is produced.
  - Error: If zero skills are discovered, the coordinator SHALL halt and report that no review skills are installed.

- **FR-004**: The coordinator SHALL dispatch discovered skills in a deterministic order. The canonical order is:
  1. `review-spec` (spec adherence)
  2. `review-security` (security)
  3. `review-quality` (code quality)
  4. `review-tests` (test quality)
  5. `review-architecture` (architecture adherence)
  6. `review-performance` (performance)
  7. `review-docs` (documentation accuracy)
  8. `review-deps` (dependency review)
  - Skills not present are skipped. Skills present but not in this list are dispatched after all known skills, in alphabetical order.

#### 4.1.3 Process Compliance (Coordinator-Owned)

- **FR-005**: Before dispatching any skill, the coordinator SHALL verify the Coder's process compliance directly:
  - [ ] The WP file contains a Spec Compliance Checklist (from Coder Step 2b) for each task.
  - [ ] All checklist items in each task are checked off.
  - [ ] Activity Log entries are present and consistent with lane transitions (`planned` -> `doing` -> `for_review`).
  - [ ] Commit history shows granular commits (not a single bulk commit for the entire WP).
  - Verdict: If the Spec Compliance Checklist is missing for any task, the coordinator SHALL record a FAIL finding for "Process Compliance" and include it in the FB-XX list. Review continues with remaining dimensions.

#### 4.1.4 Encoding Check (Coordinator-Owned)

- **FR-006**: Before dispatching any skill, the coordinator SHALL scan all files created or modified as part of the WP for UTF-8 encoding violations:
  - Em dashes (U+2014), en dashes (U+2013)
  - Smart/curly quotes (U+201C, U+201D, U+2018, U+2019)
  - Curly apostrophes (U+2019)
  - Non-breaking spaces (U+00A0)
  - Ellipsis characters (U+2026)
  - Other characters in the Unicode General Punctuation block (U+2000-U+206F) that indicate copy-paste from formatted sources
  - Verdict: Encoding violations are flagged as WARN, not FAIL.

#### 4.1.5 Skill Dispatch

- **FR-007**: The coordinator SHALL dispatch each discovered skill as a subagent invocation using `runSubagent`. Each invocation SHALL include a prompt containing:
  1. The skill file path to load (e.g., `.github/skills/review-spec/SKILL.md`)
  2. The WP identifier (e.g., `WP01`)
  3. The spec file path
  4. The output findings file path (e.g., `.sdd/reviews/WP01-feature-name/review-spec-findings.md`)
  5. An instruction to read the SKILL.md first, then discover and review code, then write findings to the specified file
  - Precondition: The `.sdd/reviews/<WP-id>/` directory exists (coordinator creates it in FR-008).
  - Error: If a subagent invocation fails (tool error, timeout), the coordinator SHALL record a WARN finding for that dimension stating "Skill dispatch failed: <error>" and continue with the next skill.

- **FR-008**: The coordinator SHALL create the directory `.sdd/reviews/<WP-id>/` (e.g., `.sdd/reviews/WP01-feature-name/`) before dispatching the first skill, where `<WP-id>` is the WP filename stem (e.g., `WP01-feature-name`).
  - Error: If the directory cannot be created, the coordinator SHALL halt and report the filesystem error.

- **FR-009**: Each skill subagent SHALL execute sequentially (one at a time, blocking). The coordinator SHALL NOT dispatch the next skill until the current skill's subagent returns.

#### 4.1.6 Findings Aggregation

- **FR-010**: After all skills have been dispatched, the coordinator SHALL read every findings file in `.sdd/reviews/<WP-id>/` and parse each finding entry.
  - Error: If a findings file is missing (skill did not write output), the coordinator SHALL record a WARN for that dimension: "Skill completed but produced no findings file."

- **FR-011**: The coordinator SHALL cross-correlate findings across skills to identify:
  1. **Duplicate findings**: The same code location flagged by two or more skills for related reasons. These SHALL be merged into a single composite finding that references all source skills.
  2. **Conflicting findings**: One skill marks a construct as acceptable while another flags it as a violation (e.g., Security PASS on input validation but Code Quality flags the same code as overly complex). These SHALL be surfaced as a composite finding with both perspectives noted and the more severe verdict preserved.
  3. **Systemic patterns**: Three or more findings of the same type across different files (e.g., missing error handling in 5 endpoints). These SHALL be grouped into a single systemic finding with all locations listed.

#### 4.1.7 Verdict and Report

- **FR-012**: The coordinator SHALL produce a verdict based on the aggregated findings:
  - **Approved**: Zero FAILs across all dimensions (coordinator-owned + skills), zero WARNs.
  - **Approved with Findings**: Zero FAILs, one or more WARNs that do not block correctness.
  - **Changes Required**: One or more FAILs in any dimension. A single FAIL in any skill is sufficient.

- **FR-013**: The coordinator SHALL write a review summary into the WP file (`.sdd/plans/WP<NN>-*.md`) under a `## Review` section at the end, containing:
  1. Reviewer identification: "Review Coordinator (v2)"
  2. Review date (ISO 8601)
  3. Verdict string
  4. Skills dispatched (list of skill names with their individual PASS/WARN/FAIL status)
  5. FB-XX checklist: one actionable item per FAIL finding, with file path, line reference (if available), requirement citation, and expected fix. Numbered sequentially (FB-01, FB-02, ...).
  6. WARN items listed separately (not in the FB-XX checklist, but documented for tracking).
  7. Statistics table: dimension-level PASS/WARN/FAIL counts.
  8. Cross-correlation findings (duplicates merged, conflicts noted, systemic patterns identified).
  - The report format SHALL match the template defined in Section 8.2.

- **FR-014**: The coordinator SHALL NOT write findings from individual skills into the WP file. Only the aggregated summary, verdict, and FB-XX items go into the WP. Detailed per-skill findings remain in `.sdd/reviews/<WP-id>/`.

#### 4.1.8 WP Lifecycle Management

- **FR-015**: The coordinator SHALL update the WP file's YAML frontmatter after writing the review:
  - `lane: done` when verdict is Approved or Approved with Findings
  - `lane: to_do` when verdict is Changes Required
  - `review_status: has_feedback` when verdict is Changes Required
  - `review_status:` removed (or empty) when verdict is Approved or Approved with Findings

- **FR-016**: The coordinator SHALL append an Activity Log entry to the WP file:
  - Approved: `YYYY-MM-DDTHH:MM:SSZ - review-coordinator - lane=done - Verdict: Approved`
  - Approved with Findings: `YYYY-MM-DDTHH:MM:SSZ - review-coordinator - lane=done - Verdict: Approved with Findings (N WARNs)`
  - Changes Required: `YYYY-MM-DDTHH:MM:SSZ - review-coordinator - lane=to_do - Verdict: Changes Required (N FAILs) -- awaiting remediation`

- **FR-017**: When all WPs referencing a spec have `lane: done`, the coordinator SHALL update the spec's `> **Status**:` from `Draft` to `Approved` and include the spec file in the review commit.

#### 4.1.9 Patterns File Curation

- **FR-018**: After every review, the coordinator SHALL update `.sdd/reviews/review-patterns.md`:
  1. **Add new patterns**: For each FAIL finding, extract a concise, actionable pattern entry describing what went wrong and how to avoid it. Each pattern SHALL have a unique identifier (PAT-NNN), a category tag, and a one-line description.
  2. **Remove resolved patterns**: If a pattern from a previous review has zero occurrences in the current review (the mistake was not repeated), mark it as resolved. Resolved patterns are moved to a `## Resolved` section at the bottom, not deleted.
  3. **Keep active patterns**: Patterns that recur remain in the `## Active Patterns` section.
  - Error: If the file does not exist, create it with the initial structure defined in Section 7.3.

- **FR-019**: The coordinator SHALL NOT add WARN-level findings to the patterns file. Only FAIL-level findings generate patterns.

#### 4.1.10 Review Round Tracking

- **FR-050**: The coordinator SHALL track and display the review round number in the WP review summary:
  - Initial review: round 1.
  - Each subsequent re-review increments the round number by 1.
  - The round number is determined by counting existing Activity Log entries from `review-coordinator` in the WP file, plus 1.
  - On re-review, the coordinator SHALL overwrite the existing `## Review` section in the WP file with the new review results (not append a second review section).

#### 4.1.11 Commit

- **FR-020**: After writing the review report, updating WP frontmatter, and curating the patterns file, the coordinator SHALL commit all modified files:
  - Always include: the WP file (`.sdd/plans/WP<NN>-*.md`)
  - Include if modified: `.sdd/reviews/review-patterns.md`
  - Include if modified: the spec file (when status changes per FR-017)
  - Include: all findings files in `.sdd/reviews/<WP-id>/`
  - Commit message: `docs(review): WP<NN> verdict <Approved|Approved with Findings|Changes Required>`
  - Files SHALL be listed explicitly in `git add` -- never use `git add .` or `git add -A`.

#### 4.1.12 Re-Review

- **FR-021**: On re-review (when a WP returns to `lane: for_review` after remediation), the coordinator SHALL:
  1. Identify which skills produced FAIL findings in the previous review (by reading findings file frontmatter for `finding_counts.fail > 0`).
  2. Identify which files were modified since the last review (via `git diff` against the last review commit).
  3. Cross-reference modified files against each skill's `files_reviewed` frontmatter field to determine which PASSing skills need re-dispatch.
  4. Re-dispatch only: (a) skills that previously FAILed, and (b) skills whose `files_reviewed` list overlaps with the set of modified files.
  4. Overwrite the previous findings files for re-dispatched skills only. Findings files for skills that are not re-dispatched SHALL be preserved.
  5. Re-aggregate all findings (preserved + new) and produce a new verdict.
  - The re-review prompt to each subagent SHALL include the previous findings file path so the skill can check whether specific issues were resolved.

- **FR-022**: If after three review rounds the same FB-XX items remain unresolved, the coordinator SHALL:
  1. Set `lane: blocked` in the WP frontmatter
  2. Append `YYYY-MM-DDTHH:MM:SSZ - review-coordinator - lane=blocked - Cycle stalled: FB-XX unresolved after 3 rounds` to the Activity Log
  3. Escalate to the user via `askQuestions`
  4. Halt -- do not dispatch further skills or produce a new verdict.

#### 4.1.13 No Pipeline Orchestration

- **FR-023**: The coordinator SHALL NOT automatically scan for other WPs to review or implement after delivering a verdict. After the commit (FR-020), the coordinator presents the verdict to the user and stops.
- **FR-024**: The coordinator SHALL NOT invoke the Coder agent, Orchestrator agent, or any other agent directly after a review. Handoffs to other agents are exposed as handoff buttons only.

#### Implementation Contract -- Review Coordinator

**Inputs**:
- WP identifier (string, optional): e.g., `"WP01"`. If omitted, coordinator scans for `lane: for_review`.
- Artifact chain: WP file, spec file, ideation brief, plan index (all resolved from WP references).

**Outputs**:
- Review summary appended to WP file under `## Review` section.
- WP frontmatter updated (`lane`, `review_status`).
- Activity Log entry appended.
- Per-skill findings files written to `.sdd/reviews/<WP-id>/`.
- Patterns file updated at `.sdd/reviews/review-patterns.md`.
- Git commit with all modified files.

**Error behaviors**:
- Missing WP: halt, list available WPs, ask user.
- Missing artifact in chain: halt, report which artifact is missing.
- Zero skills discovered: halt, report no review skills installed.
- Subagent failure: record WARN for that dimension, continue.
- Missing findings file after subagent: record WARN, continue.
- Filesystem error creating review directory: halt, report error.
- Stalled review cycle (3 rounds): set `lane: blocked`, escalate to user, halt.

---

### 4.2 Review Skills (Common Contract)

#### 4.2.1 Skill Input Contract

- **FR-025**: Every review skill SHALL accept the following inputs in its subagent prompt:
  1. `skill_path`: Path to its SKILL.md file (e.g., `.github/skills/review-spec/SKILL.md`)
  2. `wp_id`: Work package identifier (e.g., `WP01`)
  3. `spec_path`: Path to the specification file
  4. `output_path`: Path to the findings output file (e.g., `.sdd/reviews/WP01-feature-name/review-spec-findings.md`)
  5. `previous_findings_path` (optional, re-review only): Path to the previous findings file for this skill

- **FR-026**: Every review skill SHALL, upon invocation:
  1. Read its own SKILL.md file to load its checklist and review instructions.
  2. Read the specification file to understand what was required.
  3. Discover and read all implementation code relevant to its review domain.
  4. Evaluate each checklist item against the discovered code.
  5. Write structured findings to the specified output path.

#### 4.2.2 Skill Output Contract (Findings File Format)

- **FR-027**: Every review skill SHALL write its findings to the output path using the format defined in Section 7.1. Each finding SHALL contain:
  1. A unique finding ID within the skill (e.g., `SPEC-001`, `SEC-001`, `QUAL-001`)
  2. A severity: `PASS`, `WARN`, or `FAIL`
  3. A checklist item reference (which checklist item triggered this finding)
  4. A requirement reference (FR-XXX, section number, or "N/A" if the finding is checklist-driven rather than spec-driven)
  5. A file path and line range (if applicable)
  6. A description of the finding (what was found)
  7. An expected behavior or fix (what should be true)
  8. Evidence (code snippet, diff excerpt, or factual observation)

- **FR-028**: Skills SHALL NOT modify the WP file, spec file, patterns file, or any source code. Skills are read-only reviewers that produce findings files as their sole output.

- **FR-029**: Skills SHALL NOT produce findings for checklist items that are not applicable to the codebase under review. Instead, they SHALL record the item as `N/A` with a brief justification (e.g., "No database access in this WP" for database security checks).

#### Implementation Contract -- Review Skills (Common)

**Inputs** (via subagent prompt):
- `skill_path` (string, required): absolute or workspace-relative path to SKILL.md
- `wp_id` (string, required): WP identifier, e.g., `"WP01"`
- `spec_path` (string, required): path to the spec file
- `output_path` (string, required): path to write findings file
- `previous_findings_path` (string, optional): path to previous findings for re-review

**Outputs**:
- Findings file written to `output_path` in the format defined in Section 7.1.

**Error behaviors**:
- SKILL.md not found at `skill_path`: subagent reports error in its return message (coordinator records WARN per FR-007).
- Spec file not found: subagent reports error (coordinator records WARN).
- No relevant code found for this skill's domain: skill writes a findings file with zero findings and a note: "No code relevant to this skill's domain was found in this WP."
- Cannot write output file: subagent reports error (coordinator records WARN per FR-007).

---

### 4.3 Spec Adherence Skill (review-spec) -- P1

- **FR-030**: The review-spec skill SHALL evaluate every functional requirement (FR-XXX) referenced by the work package's `Spec References` section against the implementation. For each FR, the skill SHALL classify adherence as:
  - **Compliant**: The FR is fully implemented as specified.
  - **Partial**: Some aspects of the FR are implemented but others are missing or incomplete.
  - **Deviating**: The FR is implemented but behaves differently from the specification.
  - **Missing**: The FR is not implemented at all.
  - Classification of Partial, Deviating, or Missing SHALL produce a FAIL finding.

- **FR-031**: The review-spec skill SHALL verify for each FR:
  - [ ] The SHALL/SHALL NOT obligation is satisfied exactly
  - [ ] Preconditions specified in the FR are enforced in code
  - [ ] Postconditions specified in the FR are produced by the code
  - [ ] Error paths described in the FR are handled as specified
  - [ ] Edge cases documented in the spec (Section 5 acceptance scenarios) are covered
  - [ ] Data model fields, types, and validation rules match Section 7 of the spec
  - [ ] API request/response schemas match Section 8 of the spec
  - [ ] Error codes returned match the spec's error taxonomy

- **FR-032**: The review-spec skill SHALL check for stub implementations:
  - `pass`, `raise NotImplementedError`, `...`, `# TODO`, empty function bodies, or any placeholder that makes a test vacuously pass SHALL be classified as Missing, not Partial.

- **FR-033**: The review-spec skill SHALL verify success criteria (SC-XXX) referenced by the WP:
  - Each SC-XXX has evidence (passing test, observable behavior, measurable metric).
  - Evidence is not fabricated (the claimed test or metric actually exists).
  - SC-XXX that cannot be verified at this stage are documented with a reason.

---

### 4.4 Security Skill (review-security) -- P1

- **FR-034**: The review-security skill SHALL audit the implementation against all 14 OWASP Secure Coding Practices categories. For each category, the skill SHALL evaluate applicable checklist items and skip items that do not apply to the codebase (recording them as N/A with justification).

  The 14 categories and key checklist items are:

  1. **Input Validation**: Server-side validation, allow-list approach, data type/range/length checks, centralized validation routine, canonicalization, rejection on failure.
  2. **Output Encoding**: Server-side encoding, context-appropriate encoding for all untrusted data returned to clients, sanitization for SQL/XML/LDAP/OS commands.
  3. **Authentication and Password Management**: Auth required for all non-public resources, secure credential storage (salted hashes), fail-secure auth controls, no credential leakage in error messages, account lockout after failed attempts, MFA for sensitive operations.
  4. **Session Management**: Server-side session creation, sufficient randomness, inactivity timeout, new session ID on re-auth, no session IDs in URLs/logs, HttpOnly and Secure cookie flags.
  5. **Access Control**: Centralized authorization, fail-secure, enforced on every request, least privilege, RBAC/ABAC enforcement per spec.
  6. **Cryptographic Practices**: Approved algorithms only (no custom crypto), secure random number generation, proper key management, FIPS 140-2 compliance where required.
  7. **Error Handling and Logging**: No sensitive data in error responses, generic error messages to users, centralized logging, log security events (auth failures, access control failures, input validation failures), no sensitive data in logs.
  8. **Data Protection**: Least privilege data access, encrypted sensitive data at rest, no secrets in source code, no sensitive data in GET parameters, cache control for sensitive pages.
  9. **Communication Security**: TLS for all sensitive data transmission, valid certificates, no fallback to insecure connections, character encoding specified for all connections.
  10. **System Configuration**: Latest approved versions, all patches applied, unnecessary functionality removed, test code removed from production, HTTP methods restricted, security headers present.
  11. **Database Security**: Parameterized queries (no SQL injection), least-privilege DB access, no hardcoded connection strings, stored procedures for data abstraction, default credentials changed.
  12. **File Management**: No user-supplied data in dynamic includes, auth before upload, file type validation by headers (not extension), upload directory execution disabled, no user data in redirects, no absolute paths to client.
  13. **Memory Management**: Buffer size checks, null termination handling, resource cleanup (not relying on GC), no known vulnerable functions.
  14. **General Coding Practices**: No direct OS commands from user input, checksums for integrity verification, locking for race conditions, explicit variable initialization, no dynamic code execution from user data.

- **FR-035**: The review-security skill SHALL cross-reference findings against the spec's security requirements (Section 10.2) to verify that all specified security controls are implemented.

- **FR-036**: The review-security skill SHALL use web research (`#tool:web`) to verify unfamiliar security patterns against current OWASP guidelines, framework-specific security documentation, or CVE databases when encountered during review.

---

### 4.5 Code Quality Skill (review-quality) -- P1

- **FR-037**: The review-quality skill SHALL evaluate the implementation across the following dimensions:

  1. **Readability**: Code is understandable without extensive comments. Functions are concise and single-purpose. Control flow is straightforward (low nesting depth, no convoluted logic).
  2. **Complexity**: Functions with high cyclomatic complexity (branching, nested conditionals, multiple loops) are flagged. Recommended threshold: flag functions with complexity > 10.
  3. **Naming Quality**: Variables, functions, classes, and modules have descriptive, intention-revealing names. No single-letter variables outside loop counters. No misleading names (name does not match behavior). Naming is consistent with codebase conventions.
  4. **Comment Quality**: Comments explain "why", not "what". No commented-out code. No redundant comments that restate the code. TODO/FIXME/HACK markers are flagged as WARN.
  5. **Error Handling**: Errors are handled explicitly (no bare `except`, no swallowed exceptions). Error messages are descriptive. Error recovery is graceful. Error types are specific (not generic catch-all).
  6. **Style and Consistency**: Code follows the codebase's established patterns (indentation, bracket style, import ordering, module structure). No inconsistencies introduced by the WP.
  7. **Dead Code**: Declared symbols (functions, classes, variables, imports, routes) that are never referenced anywhere are flagged. Unreachable code paths are flagged.
  8. **Duplication**: Significant code duplication (3+ lines of identical or near-identical logic in multiple locations) is flagged.

- **FR-038**: The review-quality skill SHALL produce a FAIL for dead code, unreachable code, or bare exception handlers. Complexity, naming, comment, and style issues SHALL produce WARN unless they significantly impair maintainability.

- **FR-039**: The review-quality skill SHALL NOT enforce subjective style preferences not already established in the codebase. It reviews for consistency with existing patterns, not for the skill author's preferences.

---

### 4.6 Test Quality Skill (review-tests) -- P2

- **FR-040**: The review-tests skill SHALL evaluate all test files associated with the WP:

  1. **Test Validity**: Tests can actually fail. Flag tests with `assert True`, empty bodies, no assertions, or tests that mock away the entire subject under test (vacuous tests).
  2. **Coverage Thresholds**: Code coverage >= 80% for WP files; branch coverage >= 90% for WP files. Flag files below threshold. Flag `# pragma: no cover` or equivalent exclusions without documented justification.
  3. **BDD Scenario Matching**: Every acceptance scenario from the spec (Section 5 & 11.2) that maps to this WP has a corresponding test. Flag missing scenario coverage.
  4. **Edge Case Coverage**: Error paths, boundary values, empty inputs, maximum inputs, and concurrent access scenarios (where applicable) are tested.
  5. **Test Structure**: Tests follow Arrange/Act/Assert (or Given/When/Then) pattern. Tests are isolated (no shared mutable state between tests). Test naming is descriptive and matches the behavior tested.
  6. **Error Path Testing**: Every specified error response (from the spec's error taxonomy) has at least one test exercising it.

- **FR-041**: The review-tests skill SHALL produce a FAIL for: vacuous tests, coverage below threshold without justification, missing BDD scenario coverage. WARN for: test naming issues, minor structural concerns.

---

### 4.7 Architecture Skill (review-architecture) -- P2

- **FR-042**: The review-architecture skill SHALL evaluate:

  1. **Component Adherence**: Implemented components match the system design (Section 9.1 of spec).
  2. **Technology Stack Compliance**: Technologies used match Section 9.2. No unauthorized substitutions.
  3. **Directory Structure Compliance**: File locations match Section 9.3.
  4. **Key Design Decisions**: Architectural decisions from Section 9.4 are honored in implementation.
  5. **Separation of Concerns**: Each module has a single clear responsibility. No "god objects" or "god modules".
  6. **SOLID Principles**: Single Responsibility Principle especially. Flag classes/modules with multiple unrelated responsibilities.
  7. **Dependency Direction**: High-level modules do not depend on low-level implementation details. Dependencies flow in the correct direction per the architecture.
  8. **Scope Discipline**: No code outside what the WP tasks required. No files modified beyond the WP's declared scope. No unspecified features, abstractions, or utilities added.

- **FR-043**: The review-architecture skill SHALL produce a FAIL for: scope creep (unspecified code), technology stack violations, component design violations. WARN for: SRP concerns, minor structural deviations.

---

### 4.8 Performance Skill (review-performance) -- P3

- **FR-044**: The review-performance skill SHALL check for:

  1. **N+1 Query Patterns**: Loading related entities in a loop instead of a join or batch query.
  2. **Missing Database Indexes**: Frequently queried columns without indexes (based on query patterns in code).
  3. **Blocking in Async Contexts**: Synchronous blocking calls where async operations are available and expected.
  4. **Unbounded Data Fetching**: Missing pagination, limits, or streaming for queries that could return large result sets.
  5. **Unnecessary Computation in Hot Paths**: Redundant parsing, re-serialization, repeated lookups, or recomputation of values that could be cached.
  6. **Inefficient Data Structures**: Use of data structures that are algorithmically suboptimal for the access pattern (e.g., linear search in a list that should be a set/map).
  7. **Missing Caching**: Opportunities for caching expensive computations or remote calls that are called repeatedly with the same inputs.

- **FR-045**: All performance findings SHALL be WARN severity unless the issue violates a specific performance NFR from the spec (Section 10.1), in which case it SHALL be FAIL.

---

### 4.9 Documentation Skill (review-docs) -- P3

- **FR-046**: The review-docs skill SHALL compare `.sdd/docs/` content against the actual implementation:

  1. **Architecture Docs** (`architecture.md`): Module structure matches real directory layout and component relationships.
  2. **API Reference** (`api-reference.md`): Documented endpoints, parameters, response schemas, and error codes match actual implementation.
  3. **Configuration Guide** (`configuration-guide.md`): Documented environment variables, defaults, and configuration options match actual code.
  4. **Data Model Docs**: Documented entities, fields, and relationships match actual schema.
  5. **User Guide** (`user-guide.md`): Documented user flows and behaviors match actual application behavior.
  6. **Developer Guide** (`developer-guide.md`): Setup instructions, project structure, and conventions match actual state.
  7. **Deployment Guide** (`deployment-guide.md`): Prerequisites and process match actual deployment requirements.
  8. **Staleness**: No outdated content that references removed features, old APIs, or deprecated behavior.
  9. **Completeness**: All six standard doc files exist and are populated.

- **FR-047**: Missing or empty required doc files SHALL produce a FAIL. Inaccurate content SHALL produce a FAIL. Minor omissions (missing one parameter in an otherwise accurate doc) SHALL produce a WARN.

---

### 4.10 Dependencies Skill (review-deps) -- P3

- **FR-048**: The review-deps skill SHALL review all project dependencies:

  1. **Known CVEs**: Check dependencies against known vulnerability databases (use `#tool:web` to research CVEs for major dependencies).
  2. **Abandoned/Unmaintained Packages**: Flag dependencies with no commits in the last 12 months or that are explicitly archived/deprecated.
  3. **Unnecessary Dependencies**: Flag dependencies that are imported but never used, or that duplicate functionality already available in the runtime or another dependency.
  4. **License Compatibility**: Verify dependency licenses are compatible with the project's license.
  5. **Version Pinning**: Verify dependencies use exact version pins or lock files, not floating ranges (e.g., `^` or `~` without a lockfile).
  6. **Supply Chain Integrity**: Verify lockfile exists and checksums are present. Flag missing lockfiles.

- **FR-049**: Known CVEs with CVSS score >= 7.0 (High/Critical) SHALL produce a FAIL. Abandoned packages, license issues, and missing lockfiles SHALL produce a WARN. Low-severity CVEs (CVSS < 7.0) SHALL produce a WARN.

---

## 5. User Stories

### US-01 -- Run a Full Review (Priority: P1) MVP

**As the** Coder Agent, **I want** the Review Coordinator to dispatch all available review skills against my completed WP, **so that** I receive a comprehensive, multi-dimensional review with actionable findings.

**Why P1**: This is the core review flow. Without it, no review can happen.

**Independent Test**: Submit a WP with `lane: for_review` that has known spec deviations and security issues. Verify: coordinator discovers and dispatches all installed skills, each skill writes a findings file, coordinator produces an aggregated verdict with FB-XX items citing specific files and requirements.

**Acceptance Scenarios**:
1. **Given** a WP with `lane: for_review` and 3 P1 review skills installed, **When** the coordinator is invoked with the WP ID, **Then** it dispatches all 3 skills sequentially, reads all 3 findings files, produces a verdict, writes the review to the WP file, updates frontmatter, and commits.
2. **Given** a WP with known FAIL-level issues, **When** the review completes, **Then** the verdict is "Changes Required", `lane: to_do` is set, `review_status: has_feedback` is set, and the FB-XX list contains one entry per FAIL with file path and requirement citation.
3. **Given** a WP with no issues, **When** the review completes, **Then** the verdict is "Approved", `lane: done` is set, and no FB-XX items are present.

---

### US-02 -- Deep Security Audit (Priority: P1) MVP

**As a** Human Developer, **I want** every code change reviewed against the full 14-category OWASP Secure Coding Practices checklist, **so that** security vulnerabilities are caught before they reach production.

**Why P1**: Security was identified as the primary gap in the current reviewer (3 bullets vs 100+ checklist items).

**Independent Test**: Submit code with a SQL injection vulnerability, a hardcoded secret, and missing input validation. Verify: the review-security skill flags all three issues with specific OWASP category references and FAIL severity.

**Acceptance Scenarios**:
1. **Given** code that uses string concatenation in SQL queries, **When** review-security runs, **Then** it produces a FAIL finding referencing OWASP "Database Security" category with the specific file and line.
2. **Given** code with an API key hardcoded in source, **When** review-security runs, **Then** it produces a FAIL referencing OWASP "Data Protection" category.
3. **Given** code that only uses parameterized queries and environment variables for secrets, **When** review-security runs, **Then** those checklist items are marked PASS.
4. **Given** a WP that does not access a database, **When** review-security evaluates "Database Security", **Then** the category is marked N/A with justification "No database access in this WP."

---

### US-03 -- Cross-Correlation of Findings (Priority: P1) MVP

**As the** Coder Agent, **I want** duplicate findings from different skills merged into single composite items, **so that** I do not fix the same issue twice under different labels.

**Why P1**: Without cross-correlation, the Coder receives redundant, potentially conflicting feedback.

**Independent Test**: Submit code where the same function has both a security issue (eval on user input) and a code quality issue (eval is flagged as dangerous pattern). Verify: the coordinator merges both into one composite finding referencing both skills.

**Acceptance Scenarios**:
1. **Given** two skills flag the same file and line for related reasons, **When** the coordinator aggregates, **Then** a single composite FB-XX item is produced referencing both skills.
2. **Given** one skill marks code as PASS while another marks the same code as FAIL, **When** the coordinator aggregates, **Then** the FAIL severity is preserved and both perspectives are documented.
3. **Given** five or more findings of the same type across different files, **When** the coordinator aggregates, **Then** they are grouped as a systemic finding with all locations listed under one FB-XX item.

---

### US-04 -- Review Patterns Learning (Priority: P1) MVP

**As the** Coder Agent, **I want** a curated patterns checklist updated after each review, **so that** I can read it before implementing and avoid repeating previously-caught mistakes.

**Why P1**: The patterns file is the feedback mechanism between reviewer and coder across WPs.

**Independent Test**: After a review with FAIL findings, verify: `.sdd/reviews/review-patterns.md` contains new PAT-NNN entries for each FAIL. After a subsequent review where those mistakes are not repeated, verify: the patterns are moved to the Resolved section.

**Acceptance Scenarios**:
1. **Given** a review with 3 FAIL findings, **When** patterns curation runs, **Then** 3 new PAT-NNN entries are added to the Active Patterns section of `.sdd/reviews/review-patterns.md`.
2. **Given** an active pattern PAT-005 from a previous review, **When** the current review has zero findings matching that pattern, **Then** PAT-005 is moved to the Resolved section.
3. **Given** a review with only WARN findings, **When** patterns curation runs, **Then** no new patterns are added (WARNs do not generate patterns per FR-019).

---

### US-05 -- Dynamic Skill Discovery (Priority: P1) MVP

**As a** system maintainer, **I want** to add a new review skill by creating a `.github/skills/review-<name>/SKILL.md` file and have it automatically picked up by the coordinator, **so that** extending the review system does not require editing the coordinator agent.

**Why P1**: Dynamic discovery is what makes the architecture extensible and maintainable.

**Independent Test**: Add a new `.github/skills/review-foo/SKILL.md` file. Run a review. Verify: the coordinator discovers and dispatches `review-foo` alongside existing skills.

**Acceptance Scenarios**:
1. **Given** 3 skills exist in `.github/skills/review-*/`, **When** a 4th skill `review-foo` is added and a review runs, **Then** 4 skills are dispatched.
2. **Given** a skill `review-bar` exists, **When** it is deleted and a review runs, **Then** only the remaining skills are dispatched and no error occurs.
3. **Given** zero review skills exist, **When** a review is attempted, **Then** the coordinator halts with an error: "No review skills installed."

---

### US-06 -- Re-Review After Fixes (Priority: P2)

**As the** Coder Agent, **I want** re-reviews to re-dispatch only the skills that previously failed plus skills whose files were touched by my fixes, **so that** re-reviews are faster while still catching regressions.

**Why P2**: Efficiency improvement for the fix-review cycle, but initial reviews work without this.

**Independent Test**: After a review with 1 skill FAILing, fix the issues and submit for re-review. Verify: only the FAILed skill and skills whose files were modified are re-dispatched.

**Acceptance Scenarios**:
1. **Given** a previous review where review-spec FAILed and review-security PASSed, **When** the Coder fixes and re-submits, **Then** review-spec is re-dispatched and review-security is only re-dispatched if files it previously reviewed were modified.
2. **Given** a re-review where all previously-FAILed skills now PASS, **When** the coordinator aggregates, **Then** the verdict is Approved (or Approved with Findings if WARNs remain).
3. **Given** 3 review rounds with the same FB-XX unresolved, **When** the coordinator detects this, **Then** it sets `lane: blocked` and escalates to the user.

---

### US-07 -- Audit Trail (Priority: P2)

**As a** Human Developer, **I want** per-skill findings preserved in `.sdd/reviews/<WP-id>/` after every review, **so that** I can trace which skill found what, when, and what the evidence was.

**Why P2**: Important for accountability and dispute resolution, but the core review works without it.

**Independent Test**: After a review, verify: `.sdd/reviews/WP01-feature-name/` contains one findings file per dispatched skill, each with the correct structured format.

**Acceptance Scenarios**:
1. **Given** a review dispatching 3 skills, **When** the review completes, **Then** 3 findings files exist in `.sdd/reviews/<WP-id>/`.
2. **Given** a re-review dispatching 1 skill, **When** the re-review completes, **Then** only 1 findings file is overwritten; the other 2 are preserved from the initial review.

---

### Edge Cases

- What happens when a subagent times out or crashes? Coordinator records a WARN and continues (FR-007).
- What happens when a skill writes an empty findings file? Coordinator treats it as zero findings for that dimension (all PASS implied).
- What happens when the patterns file grows beyond 50 entries? No hard cap -- the curation logic (FR-018) actively removes resolved patterns to prevent unbounded growth.
- What happens when two WPs are `lane: for_review` simultaneously? Coordinator reviews only the WP specified or asks the user to choose (FR-001). It does NOT auto-continue to the second WP (FR-023).

---

## 6. User Flows

### 6.1 Initial Review Flow

1. User or Orchestrator invokes the Review Coordinator with a WP ID (or no ID).
2. Coordinator resolves WP: if ID given, load that WP; otherwise scan for `lane: for_review` WPs and ask the user to choose one.
3. Coordinator loads the full artifact chain (WP -> spec -> brief -> plan index). If any artifact is missing, halt with error.
4. Coordinator creates `.sdd/reviews/<WP-id>/` directory.
5. Coordinator runs process compliance checks (FR-005). Records findings (PASS or FAIL).
6. Coordinator runs encoding checks (FR-006). Records findings (PASS or WARN).
7. Coordinator discovers review skills by scanning `.github/skills/review-*/SKILL.md` (FR-003).
8. For each discovered skill (in canonical order per FR-004):
   a. Coordinator invokes `runSubagent` with the skill prompt (FR-007).
   b. Subagent reads SKILL.md, discovers code, reviews, writes findings file.
   c. Coordinator waits for subagent to return (FR-009).
   d. If subagent fails, coordinator records WARN and continues.
9. Coordinator reads all findings files from `.sdd/reviews/<WP-id>/` (FR-010).
10. Coordinator cross-correlates: merge duplicates, flag conflicts, group systemic patterns (FR-011).
11. Coordinator determines verdict (FR-012).
12. Coordinator writes review summary to WP file (FR-013).
13. Coordinator updates WP frontmatter: `lane` and `review_status` (FR-015).
14. Coordinator appends Activity Log entry (FR-016).
15. Coordinator curates patterns file (FR-018).
16. Coordinator commits all changes (FR-020).
17. Coordinator presents verdict and report to user. Stops. (FR-023).

### 6.2 Re-Review Flow

1. User or Orchestrator invokes coordinator with a WP that has `lane: for_review` after remediation.
2. Coordinator loads artifact chain (same as initial review).
3. Coordinator identifies previously FAILed skills (by reading existing findings files).
4. Coordinator identifies files modified since last review (via `git diff` against last review commit or activity log timestamps).
5. Coordinator determines re-dispatch set: FAILed skills + skills whose files were modified (FR-021).
6. Coordinator runs process compliance checks (FR-005).
7. Coordinator runs encoding checks (FR-006).
8. For each skill in the re-dispatch set:
   a. Coordinator invokes `runSubagent` with re-review prompt including previous findings path.
   b. Subagent reviews focused on previous FAIL items and any new code.
   c. Subagent overwrites the findings file for this skill.
9. Coordinator reads ALL findings files (preserved + overwritten) and re-aggregates (FR-010, FR-011).
10. Coordinator determines new verdict (FR-012).
11. Coordinator checks for stalled cycle: if same FB-XX items unresolved after 3 rounds, escalate (FR-022).
12. Steps 12-17 same as initial review.

### 6.3 Stalled Review Escalation Flow

1. Coordinator detects 3 rounds with same FB-XX items unresolved.
2. Coordinator sets `lane: blocked` in WP frontmatter.
3. Coordinator appends Activity Log: `lane=blocked - Cycle stalled`.
4. Coordinator commits the WP file.
5. Coordinator asks the user via `askQuestions` to decide next steps. Halts.

---

## 7. Data Model

### 7.1 Skill Findings File

**Path**: `.sdd/reviews/<WP-id>/<skill-name>-findings.md`
**Example**: `.sdd/reviews/WP01-auth-endpoints/review-security-findings.md`

```markdown
---
skill: review-security
wp: WP01
spec: .sdd/specs/001-feature-name.spec.md
reviewed_at: 2026-04-04T10:30:00Z
status: completed
finding_counts:
  pass: 12
  warn: 2
  fail: 3
  na: 5
files_reviewed:
  - src/api/users.py
  - src/api/auth.py
  - src/api/validators.py
  - src/config.py
---

# review-security Findings for WP01

## Summary

Brief overview of what was reviewed and overall assessment.

## Findings

### SEC-001 [FAIL]
- **Checklist item**: Database Security - Parameterized queries
- **Requirement**: FR-042, OWASP SCP 2.11
- **File**: src/api/users.py#L45-L52
- **Description**: SQL query constructed via string concatenation with user input.
- **Expected**: Use parameterized queries or ORM query builder.
- **Evidence**:
  ```python
  query = f"SELECT * FROM users WHERE name = '{user_input}'"
  ```

### SEC-002 [PASS]
- **Checklist item**: Input Validation - Server-side validation
- **Requirement**: FR-030
- **File**: src/api/validators.py
- **Description**: All endpoint inputs validated server-side using Pydantic models.
- **Evidence**: Every route handler validates input via dependency injection.

### SEC-003 [WARN]
- **Checklist item**: Data Protection - Cache control
- **Requirement**: OWASP SCP 2.8
- **File**: src/api/auth.py#L20
- **Description**: No Cache-Control headers set on authentication responses.
- **Expected**: Set `Cache-Control: no-store` on auth responses.
- **Evidence**: Response object returned without cache headers.

### SEC-004 [N/A]
- **Checklist item**: Memory Management - Buffer checks
- **Justification**: Python application; memory management is handled by the runtime.
```

**Entity fields**:

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| `skill` | string | yes | Must match a `review-*` skill name | The skill that produced this file |
| `wp` | string | yes | Must match `WP<NN>` pattern | Work package identifier |
| `spec` | string | yes | Valid file path | Path to the specification used |
| `reviewed_at` | string (ISO 8601) | yes | RFC 3339 format | Timestamp of review completion |
| `status` | enum | yes | `completed`, `error` | Whether the skill finished successfully |
| `finding_counts.pass` | integer | yes | >= 0 | Count of PASS findings |
| `finding_counts.warn` | integer | yes | >= 0 | Count of WARN findings |
| `finding_counts.fail` | integer | yes | >= 0 | Count of FAIL findings |
| `finding_counts.na` | integer | yes | >= 0 | Count of N/A items |
| `files_reviewed` | array(string) | yes | List of workspace-relative file paths | All files the skill read and evaluated during this review |

**Finding entry fields**:

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| Finding ID | string | yes | `<SKILL_PREFIX>-<NNN>`, e.g., `SEC-001` | Unique within the file |
| Severity | enum | yes | `PASS`, `WARN`, `FAIL`, `N/A` | Finding severity |
| Checklist item | string | yes | Free text referencing the skill's checklist | Which checklist item triggered this |
| Requirement | string | yes | `FR-XXX`, section ref, OWASP ref, or `N/A` | Spec or standard reference |
| File | string | conditional | Required for WARN/FAIL; file path with optional `#L<start>-L<end>` | Source file and line range |
| Description | string | yes | 1-500 characters | What was observed |
| Expected | string | conditional | Required for WARN/FAIL | What should be true |
| Evidence | string | conditional | Required for WARN/FAIL; code snippet, command output, or factual observation | Proof of the finding |
| Justification | string | conditional | Required for N/A | Why the checklist item does not apply |

**Validation rules**:
- Every FAIL and WARN finding MUST have File, Expected, and Evidence fields populated.
- Every N/A finding MUST have a Justification field.
- PASS findings MAY omit Expected and Evidence (Description and File are sufficient).
- Finding IDs MUST be sequential within the file (no gaps).
- `finding_counts` MUST accurately reflect the actual findings in the file.

---

### 7.2 Review Summary in WP File

**Location**: Appended to `.sdd/plans/WP<NN>-*.md` under `## Review` section.

```markdown
## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-04T10:45:00Z
> **Verdict**: Changes Required
> **Skills dispatched**: review-spec (FAIL), review-security (FAIL), review-quality (PASS)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist present and complete
- [PASS] Activity Log consistent
- [WARN] Encoding: 2 em dashes found in src/config.py

### Review Feedback

> Implementers: address every FB-XX item before returning for re-review.

- [ ] **FB-01**: [SPEC] FR-012 Missing - User registration endpoint does not return
  the created user object as specified. File: src/api/users.py#L45.
  Expected: POST /users returns 201 with full user schema.
  Source skills: review-spec (SPEC-003)
- [ ] **FB-02**: [SECURITY] OWASP Database Security - SQL injection in user lookup.
  File: src/api/users.py#L52. Expected: Use parameterized query.
  Source skills: review-security (SEC-001), review-quality (QUAL-007)
- [ ] **FB-03**: [SPEC] FR-015 Deviating - Error response format uses "error" key
  instead of specified "detail" key. File: src/api/errors.py#L10.
  Source skills: review-spec (SPEC-005)

### Warnings
- [WARN] No Cache-Control headers on auth responses (review-security SEC-003)
- [WARN] Encoding: em dashes in config comments

### Cross-Correlation Notes
- FB-02 is a composite finding: review-security flagged SQL injection,
  review-quality independently flagged the same code as using string formatting
  for queries.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 2 | 1 | 0 |
| review-spec | 8 | 0 | 3 |
| review-security | 12 | 2 | 1 |
| review-quality | 15 | 3 | 0 |
| **Total** | **37** | **6** | **4** |
```

---

### 7.3 Review Patterns File

**Path**: `.sdd/reviews/review-patterns.md`

```markdown
# Review Patterns

> Last updated: 2026-04-04T10:45:00Z
> Last review: WP01

Coder: read this file before implementing any WP. These patterns document
mistakes caught in previous reviews. Avoid repeating them.

## Active Patterns

### PAT-001 [security] SQL injection via string formatting
- **First seen**: WP01 (2026-04-04)
- **Occurrences**: 1
- **Pattern**: Using f-strings or .format() to build SQL queries with user input
- **Fix**: Always use parameterized queries (e.g., `cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))`)
- **Source**: review-security SEC-001

### PAT-002 [spec-adherence] Missing error response fields
- **First seen**: WP01 (2026-04-04)
- **Occurrences**: 2
- **Pattern**: Error responses missing required fields from spec error taxonomy
- **Fix**: Cross-reference every error response against Section 8 error codes before implementation
- **Source**: review-spec SPEC-005, SPEC-008

## Resolved

### PAT-000 [quality] Bare except clauses
- **First seen**: WP00 (2026-03-28)
- **Resolved**: WP01 (2026-04-04)
- **Pattern**: Using bare `except:` without specifying exception type
- **Fix**: Always catch specific exception types
```

**Entity fields**:

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| Pattern ID | string | yes | `PAT-<NNN>`, sequential | Unique identifier |
| Category tag | string | yes | One of: `security`, `spec-adherence`, `quality`, `tests`, `architecture`, `performance`, `docs`, `deps`, `process` | Review dimension that originated the pattern |
| Title | string | yes | 1-100 characters | Concise description of the mistake pattern |
| First seen | string | yes | `WP<NN> (YYYY-MM-DD)` | Which WP and when the pattern was first caught |
| Occurrences | integer | yes | >= 1 | Number of times this pattern has been caught across all reviews |
| Pattern | string | yes | 1-300 characters | Description of the mistake in general terms |
| Fix | string | yes | 1-300 characters | How to avoid this mistake |
| Source | string | yes | Comma-separated finding IDs (e.g., `review-security SEC-001`) | Which findings triggered this pattern |
| Resolved (resolved only) | string | conditional | `WP<NN> (YYYY-MM-DD)`, required in Resolved section | When the pattern was resolved |

**Validation rules**:
- Pattern IDs are globally unique and never reused (even after resolution).
- Active patterns have no `Resolved` field.
- Resolved patterns have a `Resolved` field.
- `Occurrences` is incremented each time the pattern recurs in a subsequent review.
- Patterns are only created from FAIL findings (FR-019).

---

### 7.4 Coordinator Agent File

**Path**: `.github/agents/review-coordinator.agent.md`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `name` | string | `"5. Review Coordinator"` | Agent display name |
| `description` | string | Trigger keywords | When this agent is invoked |
| `tools` | array(string) | Must include: `agent/runSubagent`, file ops, `search/*`, `web`, `todo`, `vscode/askQuestions` | Tools the coordinator needs |
| `handoffs` | array(object) | Each: `label`, `agent`, `prompt`, `send` | Available handoff targets |
| `argument-hint` | string | User-facing hint | What to pass as argument |

Handoffs:
1. `Fix Findings` -> `4. Coder` (send: true)
2. `Update Specification` -> `2. Spec Architect` (send: false)
3. `Revise Plan` -> `3. Planner` (send: false)

---

### 7.5 Skill File

**Path**: `.github/skills/review-<name>/SKILL.md`

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `name` | string | `review-<name>` pattern | Skill identifier |
| `description` | string | 1-500 characters | When and how this skill is used |
| `argument-hint` | string | Optional | User-facing hint |

Skill file body contains: purpose statement, checklist organized by category, severity guidance (which items are FAIL vs WARN), and the structured output format instruction.

---

## 8. API / Interface Design

### 8.1 Coordinator Invocation Interface

The coordinator is invoked as a VS Code chat agent. It is not an HTTP API.

**Invocation methods**:
1. Direct: user types `@review-coordinator WP01` or equivalent
2. Handoff: Coder agent completes a WP and transitions `lane: for_review`, Orchestrator invokes the coordinator
3. Argument: WP identifier string (e.g., `"WP01"`) or empty (coordinator scans)

**Response**: Markdown-formatted review report presented in the VS Code chat panel, plus file modifications (WP file, findings files, patterns file, commit).

### 8.2 Review Summary Template (Output to WP File)

See Section 7.2 for the exact format.

### 8.3 Skill Subagent Prompt Interface

The coordinator constructs a prompt for each subagent. The prompt is a structured string:

```
Review WP<NN> using the <skill-name> review skill.

1. Read the skill file at: <skill_path>
2. Read the specification at: <spec_path>
3. Discover and read all implementation code relevant to this skill's domain for WP<NN>.
   The WP file is at: <wp_path>
4. Evaluate each checklist item from the skill file against the discovered code.
5. Write your findings to: <output_path>
   Use the structured findings format from the skill file.
6. Return a brief summary of your findings (counts of PASS/WARN/FAIL/N/A).

Important:
- Do NOT modify any source code, the WP file, or the spec file.
- Only write to the specified output path.
- For each finding, cite the exact file path and line range.
- Mark checklist items as N/A (with justification) if they do not apply.
```

**Re-review variant** appends:
```
This is a re-review. Previous findings are at: <previous_findings_path>
Focus on:
- Whether previous FAIL items have been resolved
- Whether fixes introduced new issues
- Any regressions in previously-PASSing items
```

### 8.4 Handoff Prompt Templates

**Fix Findings** (to Coder):
```
WP<NN> has been returned with verdict: Changes Required.

The work package is at lane=to_do with review_status=has_feedback.

Feedback items requiring remediation:
<FB-XX list from WP Review section>

Please:
1. Update review_status to acknowledged in the WP frontmatter
2. Set lane=doing and append an Activity Log entry
3. Address every FB-XX item -- no skipping, deferring, or partial fixes
4. Re-run tests after each fix
5. When all FB-XX items are resolved, set lane=for_review and request a re-review
```

**Update Specification** (to Spec Architect):
```
Review of WP<NN> found specification gaps:
<list of spec issues found>
Please revise the specification to address these gaps.
```

**Revise Plan** (to Planner):
```
Review of WP<NN> found plan-level issues:
<list of plan issues found>
Please revise the work package plan.
```

---

## 9. Architecture

### 9.1 System Design

The system consists of two component types operating within the VS Code agent framework:

1. **Review Coordinator Agent** (`.github/agents/review-coordinator.agent.md`): A lightweight dispatcher that owns the review lifecycle. It does NOT perform deep code analysis itself. It handles: artifact loading, process compliance checks, encoding checks, skill discovery, subagent dispatch, findings aggregation, cross-correlation, verdict determination, WP lifecycle updates, patterns curation, and committing.

2. **Review Skill Files** (`.github/skills/review-*/SKILL.md`): Self-contained review checklists that are loaded by subagents at runtime. Each skill file contains the domain knowledge, checklist items, severity guidance, and output format for a single review dimension. Skills are executed by generic subagents that read the skill file as their instructions.

**Interaction pattern**:
```
User/Orchestrator
       |
       v
Review Coordinator Agent
       |
       |--> Process compliance check (inline)
       |--> Encoding check (inline)
       |--> Discover skills (scan .github/skills/review-*/)
       |
       |--> runSubagent(review-spec prompt)     --> writes review-spec-findings.md
       |--> runSubagent(review-security prompt) --> writes review-security-findings.md
       |--> runSubagent(review-quality prompt)  --> writes review-quality-findings.md
       |--> runSubagent(review-tests prompt)    --> writes review-tests-findings.md
       |--> ... (for each discovered skill)
       |
       |--> Read all findings files
       |--> Cross-correlate
       |--> Determine verdict
       |--> Write to WP file + patterns file
       |--> Commit
       |--> Present to user
```

### 9.2 Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Agent framework | VS Code Copilot Chat agents (`.agent.md` files) | Existing infrastructure used by all SDD pipeline agents. Provides `runSubagent`, file operations, search, web access. |
| Skill framework | VS Code Copilot Chat skills (`SKILL.md` files) | Existing infrastructure (semantic-commit skill). Skills are loaded by subagents as instruction files. |
| Data format | Markdown with YAML frontmatter | Consistent with all existing SDD artifacts (WPs, specs, briefs). Human-readable, diff-friendly, no additional tooling required. |
| Version control | Git (via terminal commands) | Existing infrastructure. Coordinator commits review artifacts using explicit `git add` + `git commit`. |
| File system | Local workspace (`.sdd/reviews/`, `.github/skills/`, `.github/agents/`) | All artifacts are local files. No external services, databases, or APIs. |

### 9.3 Directory & Module Structure

```
.github/
  agents/
    review-coordinator.agent.md    # NEW: replaces reviewer.agent.md
    reviewer.agent.md              # DEPRECATED: kept for reference, renamed to reviewer.agent.md.deprecated
  skills/
    semantic-commit/SKILL.md       # EXISTING: unchanged
    review-spec/SKILL.md           # NEW (P1): spec adherence skill
    review-security/SKILL.md       # NEW (P1): security audit skill
    review-quality/SKILL.md        # NEW (P1): code quality skill
    review-tests/SKILL.md          # NEW (P2): test quality skill
    review-architecture/SKILL.md   # NEW (P2): architecture adherence skill
    review-performance/SKILL.md    # NEW (P3): performance review skill
    review-docs/SKILL.md           # NEW (P3): documentation accuracy skill
    review-deps/SKILL.md           # NEW (P3): dependency review skill

.sdd/
  reviews/                         # NEW: review artifacts root
    review-patterns.md             # NEW: curated patterns checklist
    WP01-feature-name/             # NEW: per-WP review directory
      review-spec-findings.md
      review-security-findings.md
      review-quality-findings.md
      ...
    WP02-another-feature/
      ...
```

### 9.4 Key Design Decisions

**Decision 1: Dynamic skill discovery over hardcoded list**
- **Rationale**: User explicitly chose dynamic discovery for extensibility. The coordinator scans `.github/skills/review-*/SKILL.md` at runtime. Adding a skill = creating a directory. No coordinator edit needed.
- **Alternatives considered**: Hardcoded list (simpler but requires coordinator edits), hybrid (hardcoded known + dynamic unknown).
- **Consequences**: Skills with non-standard naming are dispatched in alphabetical order after known skills. Malformed skill files cause subagent errors (handled as WARN).

**Decision 2: Sequential subagent execution (not parallel)**
- **Rationale**: VS Code subagents run sequentially (blocking). Each subagent gets a fresh context window, which is the core value proposition -- no context competition between dimensions. User explicitly chose thoroughness over speed (20-40 min acceptable).
- **Alternatives considered**: Parallel subagents (not supported by framework), sequential skills within coordinator (context competition returns).
- **Consequences**: Total review time = sum of individual skill times. Re-review is faster (fewer skills dispatched).

**Decision 3: Remove pipeline orchestration from reviewer**
- **Rationale**: Separation of concerns. The current reviewer mixes review logic with pipeline routing (scan for next WP, hand off to Coder, auto-continue). This is the Orchestrator's job. The coordinator is a pure reviewer.
- **Alternatives considered**: Keep auto-continuation (current behavior), lightweight handoff only.
- **Consequences**: Requires Orchestrator agent update to handle post-review routing. Flagged as out-of-scope dependency.
- **Source**: Brainstorming decision log

**Decision 4: Persistent per-WP findings files (not temporary)**
- **Rationale**: Audit trail. Findings files are preserved in `.sdd/reviews/<WP-id>/` for reference, dispute resolution, and re-review scoping. Each review overwrites findings for re-dispatched skills only.
- **Alternatives considered**: Temporary directory (deleted after aggregation), in-memory via subagent return only.
- **Consequences**: Disk usage grows with WP count (mitigated: findings files are small markdown).

**Decision 5: Patterns file with active curation (not append-only)**
- **Rationale**: Prevents staleness. The coordinator actively curates: adds new patterns from FAILs, resolves patterns that stop recurring. No arbitrary cap on pattern count.
- **Alternatives considered**: Hard cap at 20 items, rolling last 5 WPs, append-only log.
- **Consequences**: Coordinator must compare current findings against existing patterns each review.

**Decision 6: Accept MVP coverage gap**
- **Rationale**: MVP (P1) delivers 3 skills (spec, security, quality) with expert-level depth. This is more valuable than shallow coverage across all 8 dimensions (the current problem). Missing dimensions (tests, architecture, performance, docs, deps) are added incrementally in P2/P3.
- **Alternatives considered**: Coordinator handles missing dimensions inline until skills exist.
- **Consequences**: No architecture, test quality, doc accuracy, performance, or dependency review during MVP. These dimensions are explicitly out of scope for MVP, not bugs.

### 9.5 External Integrations

**Git (local)**:
- Purpose: Version control for review artifacts.
- Authentication: None (local workspace).
- Key operations: `git add` (explicit file list), `git commit`, `git diff` (for re-review file change detection).
- Failure handling: If `git commit` fails, report error to user and halt. Do not retry.

**Web research (via `#tool:web`)**:
- Purpose: Security skill verifies patterns against OWASP/CVE databases. Dependencies skill checks for known CVEs.
- Authentication: None (public web).
- Key operations: Fetch OWASP guidelines, framework security docs, CVE databases, package registry info.
- Failure handling: If web fetch fails, skill records a WARN noting "Unable to verify against external source" and continues with checklist-based review.

---

## 10. Non-Functional Requirements

### 10.1 Performance

- **NFR-001**: A full initial review (3 P1 skills) SHALL complete within 30 minutes on a standard development machine with a medium-sized codebase (10-50 source files, 5000-15000 LOC).
- **NFR-002**: A re-review (1-2 skills re-dispatched) SHALL complete in under 15 minutes.
- **NFR-003**: The coordinator's own processing (process checks, encoding checks, aggregation, cross-correlation, report writing, commit) SHALL complete in under 3 minutes. The bulk of review time is in subagent execution.

### 10.2 Security

- The coordinator and skills do not handle user authentication, secrets, or sensitive data directly. They review code that may handle these things.
- **NFR-004**: Skills SHALL NOT execute any discovered code. Review is static analysis only (reading and evaluating code, not running it).
- **NFR-005**: The coordinator SHALL NOT store credentials, tokens, or API keys in any review artifact. If a skill finds a hardcoded secret during review, it SHALL reference the file and line but SHALL NOT reproduce the secret value in the findings file.
- **NFR-006**: Web research URLs SHALL only target well-known security resources (OWASP, NVD, framework docs). Skills SHALL NOT fetch arbitrary URLs from the codebase.

### 10.3 Scalability & Availability

- The system operates entirely locally within a VS Code workspace. There are no availability or horizontal scaling requirements.
- **NFR-007**: The system SHALL handle codebases up to 100 source files and 50,000 LOC without degraded review quality (though review time may increase proportionally).
- **NFR-008**: The system SHALL handle up to 20 accumulated WP review directories in `.sdd/reviews/` without performance degradation in coordinator operations (skill discovery, patterns curation, commit).

### 10.4 Accessibility

- Not applicable. The system produces markdown files consumed by other agents and viewed in VS Code. Standard VS Code accessibility features apply.

### 10.5 Observability

- **Logging**: The coordinator's Activity Log entries in the WP file serve as the primary audit trail. Each entry includes timestamp, agent identifier (`review-coordinator`), lane transition, and a description.
- **Metrics**: The Statistics table in each review report provides per-dimension PASS/WARN/FAIL counts. Trends can be observed by comparing statistics across WP reviews.
- **Alerting**: Stalled review cycles (FR-022) trigger escalation to the user. No automated alerting infrastructure is required.

---

## 11. Test Requirements

### 11.1 Unit Tests

Not applicable in the traditional sense. The "units" are agent and skill markdown files, not executable code. Validation is performed via acceptance testing and manual review.

However, the following SHALL be verified:
- Coordinator agent file is valid YAML frontmatter + valid markdown.
- Each skill file is valid YAML frontmatter + valid markdown.
- Findings file format can be parsed (frontmatter + structured markdown headings).
- Patterns file format can be parsed.

### 11.2 BDD / Acceptance Tests

These scenarios SHALL be manually verified during implementation and documented as acceptance evidence in the WP file.

```gherkin
Feature: Review Coordinator - Full Review

  Scenario: Successful initial review with all skills passing
    Given a WP "WP01" with lane "for_review"
    And 3 review skills installed (review-spec, review-security, review-quality)
    And the implementation has no issues
    When the coordinator is invoked with "WP01"
    Then all 3 skills are dispatched sequentially
    And 3 findings files are created in .sdd/reviews/WP01-*/
    And the WP file contains a Review section with verdict "Approved"
    And the WP frontmatter shows lane "done"
    And a git commit is created with message containing "verdict Approved"

  Scenario: Review with failures produces Changes Required verdict
    Given a WP "WP01" with lane "for_review"
    And the implementation has a SQL injection vulnerability
    And the implementation is missing FR-012
    When the coordinator is invoked
    Then review-security produces a FAIL finding for SQL injection
    And review-spec produces a FAIL finding for missing FR-012
    And the coordinator verdict is "Changes Required"
    And the WP frontmatter shows lane "to_do" and review_status "has_feedback"
    And the FB-XX list contains entries for both issues

  Scenario: Dynamic skill discovery
    Given 3 review skills exist in .github/skills/review-*/
    When a 4th skill "review-foo" is added to .github/skills/review-foo/SKILL.md
    And the coordinator is invoked
    Then 4 skills are dispatched

  Scenario: Zero skills installed
    Given no review-* skill directories exist in .github/skills/
    When the coordinator is invoked
    Then the coordinator halts with error "No review skills installed"

  Scenario: Subagent failure is handled gracefully
    Given a skill file that is malformed
    When the coordinator dispatches that skill
    Then a WARN finding is recorded for that dimension
    And the coordinator continues with the next skill

  Scenario: Cross-correlation merges duplicate findings
    Given review-security flags src/api/users.py#L52 as SQL injection
    And review-quality flags src/api/users.py#L52 as string formatting in query
    When the coordinator aggregates findings
    Then a single composite FB-XX item references both skills

  Scenario: Process compliance FAIL
    Given a WP where the Coder's Spec Compliance Checklist is missing
    When the coordinator checks process compliance
    Then a FAIL finding is recorded for "Process Compliance"
    And the finding is included in the FB-XX list

  Scenario: Patterns file updated after review
    Given a review with 2 FAIL findings
    When the coordinator curates the patterns file
    Then 2 new PAT-NNN entries are added to .sdd/reviews/review-patterns.md

  Scenario: Pattern resolved when mistake not repeated
    Given PAT-003 is an active pattern from a previous review
    And the current review has zero findings matching PAT-003
    When the coordinator curates the patterns file
    Then PAT-003 is moved to the Resolved section
```

```gherkin
Feature: Review Coordinator - Re-Review

  Scenario: Re-review dispatches only relevant skills
    Given a previous review where review-spec FAILed and review-security PASSed
    And the Coder has fixed the spec adherence issues
    And no files reviewed by review-security were modified
    When the coordinator re-reviews
    Then only review-spec is re-dispatched
    And review-security findings from the previous review are preserved
    And the new verdict is based on all findings (preserved + new)

  Scenario: Re-review dispatches skill when its files were modified
    Given a previous review where review-spec FAILed and review-quality PASSed
    And the Coder's fix modified files that review-quality had reviewed
    When the coordinator re-reviews
    Then both review-spec and review-quality are re-dispatched

  Scenario: Stalled review cycle escalation
    Given a WP has been reviewed 3 times
    And the same FB-01 remains unresolved in all 3 rounds
    When the coordinator detects the stall
    Then it sets lane "blocked" in the WP frontmatter
    And it escalates to the user via askQuestions
    And it halts without producing a new verdict
```

```gherkin
Feature: Security Skill - OWASP Audit

  Scenario: SQL injection detected
    Given code using string concatenation to build SQL queries
    When review-security evaluates "Database Security" checklist items
    Then a FAIL finding is produced with OWASP SCP 2.11 reference
    And the finding includes the file path and line range
    And the finding includes the vulnerable code as evidence

  Scenario: Non-applicable category skipped
    Given a WP that does not access any database
    When review-security evaluates "Database Security"
    Then the category is marked N/A with justification

  Scenario: Hardcoded secret detected
    Given code with an API key as a string literal in source
    When review-security evaluates "Data Protection"
    Then a FAIL finding is produced
    And the finding does NOT reproduce the actual secret value
```

```gherkin
Feature: Spec Adherence Skill

  Scenario: FR fully implemented
    Given FR-012 specifies a POST endpoint returning 201 with user schema
    And the implementation matches exactly
    When review-spec evaluates FR-012
    Then a PASS finding is produced

  Scenario: FR partially implemented
    Given FR-015 specifies 3 error response codes (400, 404, 409)
    And only 400 and 404 are implemented
    When review-spec evaluates FR-015
    Then a FAIL finding with status "Partial" is produced
    And the finding lists the missing 409 response

  Scenario: Stub detected as Missing
    Given a function body contains only "raise NotImplementedError"
    When review-spec evaluates the corresponding FR
    Then a FAIL finding with status "Missing" is produced
    And the classification is "Missing" not "Partial"
```

```gherkin
Feature: Code Quality Skill

  Scenario: Dead code detected
    Given a function "process_legacy()" is defined but never called anywhere
    When review-quality evaluates dead code
    Then a FAIL finding is produced listing the unused function

  Scenario: Bare except handler
    Given code uses "except:" without specifying an exception type
    When review-quality evaluates error handling
    Then a FAIL finding is produced

  Scenario: High complexity function
    Given a function has cyclomatic complexity > 10
    When review-quality evaluates complexity
    Then a WARN finding is produced with the measured complexity
```

### 11.3 Integration Tests

Integration points to verify:
- Coordinator correctly invokes `runSubagent` and receives the return value.
- Subagent correctly reads SKILL.md from the specified path.
- Subagent correctly writes findings file to the specified output path.
- Coordinator correctly reads findings files written by subagents.
- Coordinator correctly modifies WP frontmatter (YAML parsing/writing).
- Coordinator correctly appends Activity Log entries.
- Git commit includes all expected files and excludes others.

These are verified during acceptance testing (the agent framework is the integration layer).

### 11.4 End-to-End Tests

A full E2E test consists of:
1. Create a WP with `lane: for_review` and a spec with defined FRs.
2. Create implementation code with known issues (1 security, 1 spec deviation, 1 code quality).
3. Install P1 skills (review-spec, review-security, review-quality).
4. Invoke the coordinator.
5. Verify: 3 findings files created, WP updated with verdict "Changes Required", FB-XX list contains 3 items, patterns file has 3 new entries, git commit created.
6. Fix all issues.
7. Re-invoke the coordinator.
8. Verify: re-dispatch targets correct skills, new verdict is "Approved" or "Approved with Findings", patterns resolved.

### 11.5 Performance Tests

- Measure wall-clock time for a full review (3 skills) on a testbed codebase of ~5000 LOC, ~20 files. Target: < 30 minutes.
- Measure coordinator overhead (excluding subagent time). Target: < 3 minutes.

### 11.6 Security Tests

- Verify skills never execute code from the codebase (FR NFR-004).
- Verify findings files do not reproduce secret values found during review (NFR-005).
- Verify skill file frontmatter does not inject malicious tool references.

---

## 12. Constraints & Assumptions

### Constraints

- **C-001**: VS Code subagents are sequential and blocking. Parallel skill dispatch is not possible with the current framework.
- **C-002**: Each subagent has a finite context window. Skills must be designed to stay within context limits (target: < 300 lines per skill file).
- **C-003**: The coordinator agent file is a `.agent.md` markdown file with YAML frontmatter. Its behavior is defined by natural language instructions, not executable code.
- **C-004**: Skills are `.SKILL.md` markdown files. Their behavior is defined by natural language instructions and checklists, not executable code.
- **C-005**: All review artifacts are stored as local files in the workspace. No external databases or services.
- **C-006**: The existing SDD pipeline (Orchestrator -> Coder -> Reviewer cycle) expects the reviewer to set `lane:` values in WP frontmatter. This contract is preserved.

### Assumptions

- **A-001**: The VS Code `runSubagent` tool provides sufficient context window per subagent for deep review of a single dimension against a medium-sized codebase (assumption: each subagent can hold ~200K tokens).
- **A-002**: Sequential execution of 3-8 skills (20-40 minutes total) is acceptable for the target workflow. The user explicitly chose thoroughness over speed.
- **A-003**: The Coder agent will be updated separately to read `.sdd/reviews/review-patterns.md` before implementing new WPs. This is documented as an integration expectation, not implemented in this spec.
- **A-004**: The Orchestrator agent will be updated separately to handle post-review routing (scanning for next WP, invoking Coder for fixes) without relying on the reviewer's auto-continuation. This is a breaking change flagged as an out-of-scope dependency.
- **A-005**: Skills are language-agnostic. They contain review principles and checklists applicable to any programming language. Language-specific extensions are out of scope.
- **A-006**: Each subagent independently discovers and reads relevant code files using search and file-reading tools. The coordinator does not pre-read code or pass file lists to subagents.

---

## 13. Out of Scope

- **Modifying the Coder agent** to read the patterns file. Documented as integration expectation (A-003). Will be implemented in a separate effort.
- **Updating the Orchestrator agent** to remove dependency on reviewer's auto-continuation. Flagged as breaking change dependency (A-004). Required before the old reviewer can be fully retired.
- **Modifying the Planner or Spec Architect agents**. Only the reviewer is being redesigned.
- **Language-specific review rules**. Skills are language-agnostic.
- **CI/CD integration**. Review happens within the VS Code agent framework, not as a CI pipeline step.
- **Parallel skill execution**. Not supported by the VS Code subagent framework (C-001).
- **Automated code fixing**. The reviewer finds issues; the Coder fixes them.
- **Interactive hunk-level review**. The system reviews complete files, not individual diff hunks.
- **Review of non-code artifacts** (images, binary files, third-party vendored code).

---

## 14. Open Questions

- **OQ-001**: How should the Coder agent's instructions reference the patterns file?
  - Impact: Without this, the patterns file exists but the Coder may not read it, defeating the feedback loop.
  - Owner: To be resolved when the Coder agent is updated (separate effort per A-003).

- **OQ-002**: Should the Orchestrator be updated before or simultaneously with the reviewer redesign?
  - Impact: If the coordinator is deployed without an Orchestrator update, the pipeline will lose auto-continuation after reviews. The Orchestrator currently relies on the reviewer to scan for next WPs.
  - Owner: Pipeline team. Recommended: update simultaneously.

- **OQ-003**: What is the optimal skill file size?
  - Impact: Too large = context window pressure. Too small = insufficient guidance.
  - Owner: Resolved by implementation. Target: 100-300 lines per skill. Will be validated during P1 skill authoring.

- **OQ-004**: How should the re-review prompt differ from the initial review prompt?
  - Impact: A re-review subagent that ignores previous findings may re-flag resolved issues.
  - Owner: Resolved in spec. See FR-021 and Section 8.3 re-review variant prompt.

- **OQ-005**: How should the coordinator handle a brand-new skill with an unexpected findings format?
  - Impact: If a newly added skill writes findings in a non-standard format, the coordinator's aggregation may fail.
  - Owner: Mitigated by FR-027 (skill output contract). Skills that deviate from the format will produce aggregation warnings but will not crash the coordinator.

---

## 15. Glossary

| Term | Definition |
|------|-----------|
| **Coordinator** | The Review Coordinator agent (`review-coordinator.agent.md`) that dispatches skills and aggregates findings. |
| **Skill** | A self-contained review checklist file (`SKILL.md`) focused on a single review dimension. |
| **Subagent** | A VS Code agent instance spawned by `runSubagent` to execute a single skill in a fresh context window. |
| **Finding** | A single observation from a skill review, classified as PASS, WARN, FAIL, or N/A. |
| **FB-XX** | A feedback item in the WP review report. Numbered sequentially (FB-01, FB-02, ...). Each FB-XX corresponds to one or more FAIL findings. |
| **PAT-NNN** | A pattern entry in the review patterns file. Represents a recurring mistake type. |
| **Verdict** | The overall review result: Approved, Approved with Findings, or Changes Required. |
| **Cross-correlation** | The coordinator's process of merging duplicate findings, surfacing conflicts, and grouping systemic patterns across skills. |
| **WP** | Work Package. A unit of implementation work defined in `.sdd/plans/WP<NN>-*.md`. |
| **Lane** | A WP lifecycle state: `planned`, `doing`, `for_review`, `done`, `to_do`, `blocked`. |
| **OWASP SCP** | OWASP Secure Coding Practices Quick Reference Guide. The 14-category security checklist used by the review-security skill. |
| **BDD** | Behavior-Driven Development. Test approach using Given/When/Then scenarios. |
| **SDD** | Spec-Driven Development. The pipeline methodology used by this project. |

---

## 16. Traceability Matrix

| FR ID | Requirement Summary | User Story | Acceptance Scenario | Test Type | Test Section Ref |
|-------|---------------------|------------|--------------------|-----------|-----------------:|
| FR-001 | Coordinator accepts WP ID or scans for for_review | US-01 | US-01 S1 | BDD | 11.2 |
| FR-002 | Load full artifact chain | US-01 | US-01 S1 | BDD | 11.2 |
| FR-003 | Dynamic skill discovery | US-05 | US-05 S1, S2, S3 | BDD | 11.2 |
| FR-004 | Deterministic skill dispatch order | US-01, US-05 | US-01 S1, US-05 S1 | BDD | 11.2 |
| FR-005 | Process compliance verification | US-01 | Process compliance FAIL scenario | BDD | 11.2 |
| FR-006 | Encoding check | US-01 | US-01 S1 | BDD | 11.2 |
| FR-007 | Skill dispatch via runSubagent | US-01, US-05 | US-01 S1, Subagent failure scenario | BDD | 11.2 |
| FR-008 | Create review directory | US-07 | US-07 S1 | BDD | 11.2 |
| FR-009 | Sequential skill execution | US-01 | US-01 S1 | BDD | 11.2 |
| FR-010 | Read all findings files | US-01, US-07 | US-01 S1 | BDD | 11.2 |
| FR-011 | Cross-correlation of findings | US-03 | US-03 S1, S2, S3 | BDD | 11.2 |
| FR-012 | Verdict determination | US-01 | US-01 S2, S3 | BDD | 11.2 |
| FR-013 | Write review summary to WP | US-01 | US-01 S1, S2 | BDD | 11.2 |
| FR-014 | Detailed findings stay in review dir | US-07 | US-07 S1 | BDD | 11.2 |
| FR-015 | Update WP frontmatter | US-01 | US-01 S2, S3 | BDD | 11.2 |
| FR-016 | Append Activity Log entry | US-01 | US-01 S1 | BDD | 11.2 |
| FR-017 | Update spec status when all WPs done | US-01 | US-01 S3 | BDD | 11.2 |
| FR-018 | Patterns file curation - add new | US-04 | US-04 S1 | BDD | 11.2 |
| FR-019 | No patterns from WARNs | US-04 | US-04 S3 | BDD | 11.2 |
| FR-020 | Commit all review artifacts | US-01 | US-01 S1 | BDD | 11.2 |
| FR-021 | Re-review scoped dispatch | US-06 | US-06 S1, S2 | BDD | 11.2 |
| FR-022 | Stalled cycle escalation | US-06 | US-06 S3 | BDD | 11.2 |
| FR-023 | No auto-continuation | US-01 | US-01 S1 | BDD | 11.2 |
| FR-024 | No direct agent invocation | US-01 | US-01 S1 | BDD | 11.2 |
| FR-025 | Skill input contract | US-01, US-02 | US-01 S1, US-02 S1 | BDD | 11.2 |
| FR-026 | Skill execution steps | US-01, US-02 | US-01 S1, US-02 S1 | BDD | 11.2 |
| FR-027 | Skill output contract (findings format) | US-07 | US-07 S1 | BDD | 11.2 |
| FR-028 | Skills are read-only | US-01 | US-01 S1 | BDD, security | 11.2, 11.6 |
| FR-029 | N/A items with justification | US-02 | US-02 S4 | BDD | 11.2 |
| FR-030 | Spec adherence - FR classification | US-01 | Spec adherence scenarios | BDD | 11.2 |
| FR-031 | Spec adherence - detailed checks | US-01 | Spec adherence scenarios | BDD | 11.2 |
| FR-032 | Stub detection | US-01 | Stub detected scenario | BDD | 11.2 |
| FR-033 | Success criteria verification | US-01 | Spec adherence scenarios | BDD | 11.2 |
| FR-034 | Security - 14 OWASP categories | US-02 | US-02 S1, S2, S3, S4 | BDD | 11.2 |
| FR-035 | Security - spec cross-reference | US-02 | US-02 S1 | BDD | 11.2 |
| FR-036 | Security - web research | US-02 | US-02 S1 | BDD | 11.2 |
| FR-037 | Code quality - 8 dimensions | US-01 | Code quality scenarios | BDD | 11.2 |
| FR-038 | Code quality - severity rules | US-01 | Code quality scenarios | BDD | 11.2 |
| FR-039 | Code quality - no subjective preferences | US-01 | Code quality scenarios | BDD | 11.2 |
| FR-040 | Test quality - 6 dimensions | US-01 | US-01 S1 | BDD | 11.2 |
| FR-041 | Test quality - severity rules | US-01 | US-01 S1 | BDD | 11.2 |
| FR-042 | Architecture - 8 dimensions | US-01 | US-01 S1 | BDD | 11.2 |
| FR-043 | Architecture - severity rules | US-01 | US-01 S1 | BDD | 11.2 |
| FR-044 | Performance - 7 check categories | US-01 | US-01 S1 | BDD | 11.2 |
| FR-045 | Performance - severity rules | US-01 | US-01 S1 | BDD | 11.2 |
| FR-046 | Documentation - 9 check categories | US-01 | US-01 S1 | BDD | 11.2 |
| FR-047 | Documentation - severity rules | US-01 | US-01 S1 | BDD | 11.2 |
| FR-048 | Dependencies - 6 check categories | US-01 | US-01 S1 | BDD | 11.2 |
| FR-049 | Dependencies - severity rules | US-01 | US-01 S1 | BDD | 11.2 |
| FR-050 | Review round tracking | US-06 | US-06 S2 | BDD | 11.2 |

**Validation**: Every FR maps to at least one US. Every US maps to at least one acceptance scenario. Every acceptance scenario maps to a test section reference.

---

## 17. Technical References

### Architecture & Patterns

- Google Engineering Practices - Code Review Developer Guide. https://google.github.io/eng-practices/review/ (Consulted 2026-04-04). Informed the review dimension taxonomy (design, functionality, complexity, tests, naming, comments, style, docs, context).
- Microsoft Architecture Center - Microservices Design Patterns. Informed the coordinator-dispatcher pattern and separation of concerns decision.
- Gergely Orosz - "Good Code Reviews, Better Code Reviews". Informed the distinction between surface-level and deep contextual review.

### Technology Stack

- VS Code Copilot Chat Extensibility - Agent and Skill `.md` file formats. https://code.visualstudio.com/docs/copilot/copilot-extensibility-overview (Consulted 2026-04-04).
- Conventional Commits v1.0.0 - Commit message format standard. https://www.conventionalcommits.org/en/v1.0.0/ (Consulted 2026-04-04).

### Security

- OWASP Secure Coding Practices Quick Reference Guide v2.1 - Full 14-category checklist. https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/stable-en/02-checklist/05-checklist.html (Consulted 2026-04-04). Defines the security skill's complete checklist.
- OWASP Developer Guide - Migrated content from the Secure Coding Practices project. https://owasp.org/www-project-developer-guide/ (Consulted 2026-04-04).

### Standards & Specifications

- YAML 1.2 Specification - Frontmatter format for agent, skill, WP, and findings files. https://yaml.org/spec/1.2.2/ (Consulted 2026-04-04).
- ISO 8601 - Date-time format for timestamps in activity logs and findings. https://www.iso.org/iso-8601-date-and-time-format.html (Consulted 2026-04-04).

---

## 18. Version History

| Version | Date | Author | Summary of Changes |
|---------|------|--------|--------------------|
| 1.0 | 2026-04-04 | Spec Architect | Initial specification from brainstorming brief. Self-review corrections: added N/A handling for skills (FR-029), clarified re-review prompt template (Section 8.3), added secret non-reproduction constraint (NFR-005), resolved OQ-004 inline, added FR-050 review round tracking, added files_reviewed field to findings frontmatter, fixed encoding check specificity (replaced vague 'other non-ASCII' with Unicode block range), corrected traceability matrix (P2/P3 skill FRs map to US-01 via dynamic dispatch), specified re-review overwrites existing Review section. |
