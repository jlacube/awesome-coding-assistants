---
skill: review-spec
wp: WP07-p3-skills
spec: .sdd/specs/001-reviewer-v2-skill-based-architecture.spec.md
reviewed_at: 2026-04-05T12:00:00Z
status: completed
finding_counts:
  pass: 21
  warn: 0
  fail: 1
  na: 4
files_reviewed:
  - .github/skills/review-performance/SKILL.md
  - .github/skills/review-docs/SKILL.md
  - .github/skills/review-deps/SKILL.md
  - .sdd/plans/WP07-p3-skills.md
  - .sdd/specs/001-reviewer-v2-skill-based-architecture.spec.md
---

# review-spec Findings for WP07-p3-skills

## Summary

Evaluated 11 functional requirements (FR-025 through FR-029, FR-044 through FR-049) plus Section 7.1 (findings format) and Section 7.5 (skill file metadata) across three deliverables: review-performance, review-docs, and review-deps SKILL.md files. 21 checks passed, 1 failed, 4 are not applicable. The single FAIL is review-deps omitting the specification file read from its input contract, deviating from FR-026 step 2. All other requirements are fully satisfied across all three skills.

## Findings

### SPEC-001 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-025 (common input contract)
- **File**: .github/skills/review-performance/SKILL.md, .github/skills/review-docs/SKILL.md, .github/skills/review-deps/SKILL.md
- **Description**: All three skills describe accepting the required inputs (skill_path, wp_id, spec_path, output_path) through the subagent prompt pattern. Each skill's input contract describes actions consistent with receiving these inputs, and the output format templates reference wp and spec_path in YAML frontmatter. The optional `previous_findings_path` is handled implicitly (same pattern as existing P1 skills).

### SPEC-002 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-026 (common execution steps)
- **File**: .github/skills/review-performance/SKILL.md
- **Description**: review-performance input contract covers all 5 required steps: (1) reads own SKILL.md, (2) reads specification file for performance NFRs (Section 10.1), (3) discovers and reads implementation code relevant to WP, (4) evaluates each checklist item, (5) writes structured findings to output path.

### SPEC-003 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-026 (common execution steps)
- **File**: .github/skills/review-docs/SKILL.md
- **Description**: review-docs input contract covers all 5 required steps: (1) reads own SKILL.md, (2) reads specification file for documentation requirements, (3) reads WP file and discovers implementation code, (4) evaluates each checklist item, (5) writes structured findings to output path.

### SPEC-004 [FAIL]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-026 (common execution steps, step 2)
- **File**: .github/skills/review-deps/SKILL.md#L8-L14
- **Description**: review-deps input contract omits FR-026 step 2: "Read the specification file to understand what was required." The input contract jumps from reading SKILL.md (step 1) directly to identifying dependency manifest files (step 2), skipping the spec file read entirely. The specification may contain dependency-relevant requirements (e.g., NFRs about specific library versions, security constraints on dependencies, or license requirements) that the skill would miss.
- **Expected**: Input contract step 2 should read: "Read the specification file to understand dependency-relevant requirements (NFRs, security constraints, license requirements)." — matching the pattern used by review-performance ("Read the specification file for any performance NFRs") and review-docs ("Read the specification file for documentation requirements").
- **Evidence**:
  ```markdown
  **Input contract** (received via subagent prompt):
  1. Read this SKILL.md file for review instructions.
  2. Identify the project's dependency manifest files (see known patterns below).
  3. For each major dependency, use `#tool:web` to research known CVEs against trusted databases.
  4. Evaluate each checklist item below.
  5. Write structured findings to the specified output path.
  6. Return a brief summary (counts of PASS/WARN/FAIL/N/A).
  ```

### SPEC-005 [PASS]
- **Checklist item**: FR classification - SHALL obligation (output format)
- **Requirement**: FR-027 (skill output contract, Section 7.1)
- **File**: .github/skills/review-performance/SKILL.md
- **Description**: review-performance output format includes all required Section 7.1 fields: YAML frontmatter with skill, wp, spec, reviewed_at, status, finding_counts (pass/warn/fail/na), and files_reviewed. Finding examples include the required fields per severity level (WARN: checklist item, requirement, file, description, expected, evidence; N/A: checklist item, justification). Finding prefix is `PERF-` with sequential numbering.

### SPEC-006 [PASS]
- **Checklist item**: FR classification - SHALL obligation (output format)
- **Requirement**: FR-027 (skill output contract, Section 7.1)
- **File**: .github/skills/review-docs/SKILL.md
- **Description**: review-docs output format includes all required Section 7.1 fields. Finding examples demonstrate FAIL, WARN, and N/A findings with all required fields per severity. Finding prefix is `DOC-` with sequential numbering. The files_reviewed list in the template pre-populates the 6 standard doc files.

### SPEC-007 [PASS]
- **Checklist item**: FR classification - SHALL obligation (output format)
- **Requirement**: FR-027 (skill output contract, Section 7.1)
- **File**: .github/skills/review-deps/SKILL.md
- **Description**: review-deps output format includes all required Section 7.1 fields. Finding examples demonstrate FAIL, WARN, and N/A findings with all required fields per severity. Finding prefix is `DEP-` with sequential numbering.

### SPEC-008 [PASS]
- **Checklist item**: FR classification - SHALL obligation (read-only constraint)
- **Requirement**: FR-028
- **File**: .github/skills/review-performance/SKILL.md, .github/skills/review-docs/SKILL.md, .github/skills/review-deps/SKILL.md
- **Description**: All three skills include an explicit read-only constraint statement referencing FR-028. review-performance: "Do NOT modify any source code, the WP file, or the spec file." review-docs: "Do NOT modify any documentation files, source code, the WP file, or the spec file." review-deps: "Do NOT modify any source code, dependency files, the WP file, or the spec file."

### SPEC-009 [PASS]
- **Checklist item**: FR classification - SHALL obligation (N/A with justification)
- **Requirement**: FR-029
- **File**: .github/skills/review-performance/SKILL.md, .github/skills/review-docs/SKILL.md, .github/skills/review-deps/SKILL.md
- **Description**: All three skills include N/A guidance with example justifications. review-performance: "No database access in this WP", "No async code in this WP". review-docs: "No API endpoints in this WP", "No environment variables introduced". review-deps: "No dependency manifest found in this project", "Project has no runtime dependencies." Additionally, review-deps handles the complete-skill-N/A edge case: "If no recognized dependency manifest is found, mark the entire skill as N/A."

### SPEC-010 [PASS]
- **Checklist item**: FR classification - SHALL obligation (7 performance categories)
- **Requirement**: FR-044
- **File**: .github/skills/review-performance/SKILL.md
- **Description**: All 7 categories from FR-044 are present with correct names and at least 3 verifiable checklist items each: (1) N+1 Query Patterns — 3 items, (2) Missing Database Indexes — 3 items, (3) Blocking in Async Contexts — 3 items, (4) Unbounded Data Fetching — 3 items, (5) Unnecessary Computation in Hot Paths — 3 items, (6) Inefficient Data Structures — 3 items, (7) Missing Caching — 3 items. Each item is phrased as a question as specified by WP acceptance criteria.

### SPEC-011 [PASS]
- **Checklist item**: FR classification - SHALL obligation (performance severity)
- **Requirement**: FR-045
- **File**: .github/skills/review-performance/SKILL.md
- **Description**: Severity guidance correctly specifies WARN as default for all performance findings and FAIL only when violating a specific performance NFR from spec Section 10.1. Includes instruction to cite the specific NFR when issuing FAIL (e.g., "Violates NFR-001: 30-minute review time").

### SPEC-012 [PASS]
- **Checklist item**: FR classification - SHALL obligation (9 documentation categories)
- **Requirement**: FR-046
- **File**: .github/skills/review-docs/SKILL.md
- **Description**: All 9 categories from FR-046 are present: (1) Architecture Docs — 4 items, (2) API Reference — 5 items, (3) Configuration Guide — 4 items, (4) Data Model Docs — 3 items, (5) User Guide — 3 items, (6) Developer Guide — 4 items, (7) Deployment Guide — 3 items, (8) Staleness — 4 items, (9) Completeness — 3 items. Each category verifies content against actual implementation, not just file existence. The 6 standard doc files are correctly listed.

### SPEC-013 [PASS]
- **Checklist item**: FR classification - SHALL obligation (documentation severity)
- **Requirement**: FR-047
- **File**: .github/skills/review-docs/SKILL.md
- **Description**: Severity guidance correctly specifies FAIL for missing/empty required doc files and inaccurate content. WARN for minor omissions. Matches FR-047 exactly.

### SPEC-014 [PASS]
- **Checklist item**: FR classification - SHALL obligation (6 dependency categories)
- **Requirement**: FR-048
- **File**: .github/skills/review-deps/SKILL.md
- **Description**: All 6 categories from FR-048 are present: (1) Known CVEs — 5 items with trusted source list and `#tool:web` instruction, (2) Abandoned/Unmaintained — 3 items with 12-month threshold, (3) Unnecessary Dependencies — 3 items, (4) License Compatibility — 4 items, (5) Version Pinning — 3 items, (6) Supply Chain Integrity — 3 items. Trusted sources listed per NFR-006. Graceful degradation for failed web research documented.

### SPEC-015 [PASS]
- **Checklist item**: FR classification - SHALL obligation (dependency severity)
- **Requirement**: FR-049
- **File**: .github/skills/review-deps/SKILL.md
- **Description**: Severity guidance correctly specifies FAIL for CVEs with CVSS >= 7.0 (High/Critical). WARN for low-severity CVEs (CVSS < 7.0), abandoned packages, license issues, missing lockfiles, unnecessary dependencies, and floating version ranges. Matches FR-049 exactly.

### SPEC-016 [PASS]
- **Checklist item**: Data model match - Section 7.1 entity fields
- **Requirement**: Section 7.1 (Skill Findings File)
- **File**: .github/skills/review-performance/SKILL.md
- **Description**: Output format template includes all required entity fields from Section 7.1: skill, wp, spec, reviewed_at, status, finding_counts (pass/warn/fail/na), files_reviewed. Finding entries include all required fields per severity level as defined in Section 7.1 validation rules.

### SPEC-017 [PASS]
- **Checklist item**: Data model match - Section 7.1 entity fields
- **Requirement**: Section 7.1 (Skill Findings File)
- **File**: .github/skills/review-docs/SKILL.md
- **Description**: Output format template includes all required entity fields from Section 7.1. Finding entries demonstrate FAIL, WARN, and N/A with appropriate required fields per Section 7.1 validation rules.

### SPEC-018 [PASS]
- **Checklist item**: Data model match - Section 7.1 entity fields
- **Requirement**: Section 7.1 (Skill Findings File)
- **File**: .github/skills/review-deps/SKILL.md
- **Description**: Output format template includes all required entity fields from Section 7.1. Finding entries demonstrate FAIL, WARN, and N/A with appropriate required fields per Section 7.1 validation rules.

### SPEC-019 [PASS]
- **Checklist item**: Data model match - Section 7.5 skill file metadata
- **Requirement**: Section 7.5 (Skill File)
- **File**: .github/skills/review-performance/SKILL.md
- **Description**: YAML frontmatter contains `name: review-performance` (matches `review-<name>` pattern), `description` (within 500 chars), and `argument-hint`. Skill file body contains purpose statement, checklist by category, severity guidance, and output format — matching Section 7.5 body structure requirement.

### SPEC-020 [PASS]
- **Checklist item**: Data model match - Section 7.5 skill file metadata
- **Requirement**: Section 7.5 (Skill File)
- **File**: .github/skills/review-docs/SKILL.md
- **Description**: YAML frontmatter contains `name: review-docs`, `description` (within 500 chars), and `argument-hint`. Skill file body contains purpose statement, checklist by category, severity guidance, and output format.

### SPEC-021 [PASS]
- **Checklist item**: Data model match - Section 7.5 skill file metadata
- **Requirement**: Section 7.5 (Skill File)
- **File**: .github/skills/review-deps/SKILL.md
- **Description**: YAML frontmatter contains `name: review-deps`, `description` (within 500 chars), and `argument-hint`. Skill file body contains purpose statement, checklist by category, severity guidance, and output format. Additionally includes known manifest/lockfile patterns and edge case handling.

### SPEC-022 [PASS]
- **Checklist item**: SC verification - SC-003 (self-contained skills under 300 lines)
- **Requirement**: SC-003
- **File**: .github/skills/review-performance/SKILL.md, .github/skills/review-docs/SKILL.md, .github/skills/review-deps/SKILL.md
- **Description**: All three skills are well under 300 lines (approximately 120, 130, and 150 lines respectively). Each is self-contained with no cross-skill dependencies or imports.

### SPEC-023 [N/A]
- **Checklist item**: Preconditions enforced
- **Justification**: Deliverables are markdown instruction files (SKILL.md), not executable code with runtime preconditions. Precondition enforcement is handled by the coordinator (FR-007) when constructing subagent prompts.

### SPEC-024 [N/A]
- **Checklist item**: Error paths handled
- **Justification**: Error handling for subagent failures is owned by the coordinator (FR-007). The skill files describe graceful behavior for domain-specific edge cases (e.g., review-deps: "If no recognized dependency manifest is found, mark the entire skill as N/A"; "If web research fails or is unavailable, record WARN") but runtime error paths are coordinator-level concerns.

### SPEC-025 [N/A]
- **Checklist item**: API contract match
- **Justification**: No API endpoints in this WP. All deliverables are markdown skill files invoked via subagent prompts, not HTTP APIs.

### SPEC-026 [N/A]
- **Checklist item**: SC verification - WP integration test
- **Justification**: Deferred verification. The WP's independent test ("Install all 8 review skills. Invoke the coordinator on a WP. Verify: coordinator discovers and dispatches all 8 skills in canonical order") requires runtime coordinator behavior across all 8 skills and cannot be verified by static review of 3 skill files alone.
