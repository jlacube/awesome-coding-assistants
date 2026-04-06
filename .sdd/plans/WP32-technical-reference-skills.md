---
lane: done
---

# WP32 - Technical Reference Doc Skills

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/007-docs-agent.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP30, WP31 |
| Goal | Implement doc-architecture and doc-api-reference skills for technical reference documentation |
| Status | Not Started |
| Independent Test | Invoke each skill with a sample WP and verify architecture.md and api-reference.md are updated correctly |
| Parallelisable | Yes |
| Prompt | `.sdd/plans/WP32-technical-reference-skills.md` |

## Objective

Implement the doc-architecture and doc-api-reference skills that produce technical reference documentation. doc-architecture generates/updates the system architecture overview from the spec and codebase. doc-api-reference generates/updates API endpoint documentation from contract files. These are the "source of truth" documentation skills that bridge specs and contracts to readable docs.

## Spec References

FR-010, FR-011, FR-012, FR-013, US-01, US-02, Section 4.2.1, Section 4.2.2, Section 7.1

## Tasks

### T32-01 - Create doc-architecture SKILL.md structure

- **Description**: Replace the stub content in `.github/skills/doc-architecture/SKILL.md` with the full skill file including YAML frontmatter, input contract (referencing DOC-SKILL-CONTRACT.md), output contract, and execution sequence sections.
- **Spec refs**: FR-010, Section 4.2.1
- **Parallel**: No
- **Acceptance criteria**:
  - [x] doc-architecture SKILL.md has valid YAML frontmatter with `name: doc-architecture`
  - [x] Input contract references DOC-SKILL-CONTRACT.md and lists all 6 context items from FR-005
  - [x] Output contract specifies `.sdd/docs/architecture.md` as the target file
  - [x] Execution sequence defines the step-by-step process for generating architecture docs
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Reference: `.github/skills/review-spec/SKILL.md` for skill file structure (frontmatter, contract, steps)
  - Pattern: Follow the common doc skill contract defined in DOC-SKILL-CONTRACT.md (WP30 T30-03)
  - Files to modify: `.github/skills/doc-architecture/SKILL.md`

### T32-02 - Write architecture docs generation sections

- **Description**: Write the skill instructions for generating architecture documentation content. The skill SHALL produce/update `.sdd/docs/architecture.md` with: system overview (from spec Section 9.1), component diagram (Mermaid or prose), technology stack summary (from spec Section 9.2), key design decisions (from spec Section 9.4), directory structure (from actual codebase, not spec), and data flow descriptions.
- **Spec refs**: FR-010
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL produce/update `.sdd/docs/architecture.md` with system overview from spec Section 9.1 (FR-010.1)
  - [x] The skill SHALL include a component diagram in Mermaid or prose format (FR-010.2)
  - [x] The skill SHALL include technology stack summary from spec Section 9.2 (FR-010.3)
  - [x] The skill SHALL include key design decisions from spec Section 9.4 (FR-010.4)
  - [x] The skill SHALL include directory structure from the actual codebase, not the spec (FR-010.5)
  - [x] The skill SHALL include data flow descriptions (FR-010.6)
- **Test requirements**: BDD
- **Depends on**: T32-01
- **Implementation Guidance**:
  - Pattern: Use `list_dir` and `file_search` to discover actual directory structure rather than copying from spec
  - Known pitfalls: FR-010.5 explicitly requires directory structure from the actual codebase -- do not just copy the spec's proposed structure
  - Files to modify: `.github/skills/doc-architecture/SKILL.md`

### T32-03 - Write incremental update logic

- **Description**: Write the skill instructions for incremental updates. On subsequent runs (when architecture.md already has content), the skill SHALL update only the sections affected by the current WP without overwriting unrelated sections.
- **Spec refs**: FR-011
- **Parallel**: No
- **Acceptance criteria**:
  - [x] On incremental updates, the skill SHALL update affected sections without overwriting unrelated sections (FR-011)
  - [x] The skill reads existing architecture.md content before writing
  - [x] Sections not affected by the current WP remain unchanged
  - [x] BDD: Given architecture.md has existing content from WP01, When doc-architecture runs for WP02, Then WP01 content is preserved and WP02 content is added/updated
- **Test requirements**: BDD
- **Depends on**: T32-02
- **Implementation Guidance**:
  - Pattern: Read existing file, identify sections by heading, merge updates into affected sections only
  - Known pitfalls: Naive overwrite destroys prior WP content -- always read before write
  - Files to modify: `.github/skills/doc-architecture/SKILL.md`

### T32-04 - Create doc-api-reference SKILL.md structure

- **Description**: Replace the stub content in `.github/skills/doc-api-reference/SKILL.md` with the full skill file including YAML frontmatter, input contract (referencing DOC-SKILL-CONTRACT.md), output contract, and execution sequence sections.
- **Spec refs**: FR-012, Section 4.2.2
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] doc-api-reference SKILL.md has valid YAML frontmatter with `name: doc-api-reference`
  - [x] Input contract references DOC-SKILL-CONTRACT.md and lists all 6 context items from FR-005
  - [x] Output contract specifies `.sdd/docs/api-reference.md` as the target file
  - [x] Execution sequence defines the step-by-step process for generating API reference docs
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Reference: `.github/skills/review-spec/SKILL.md` for skill file structure
  - Pattern: Follow DOC-SKILL-CONTRACT.md
  - Files to modify: `.github/skills/doc-api-reference/SKILL.md`

### T32-05 - Write API reference generation from contracts

- **Description**: Write the skill instructions for generating API reference documentation from contract files. The skill SHALL produce/update `.sdd/docs/api-reference.md` with: one section per API endpoint, method/path/description, request parameters and body (from contracts), response schema (from contracts), error codes and meanings (from error catalog contract), authentication requirements, and example request/response generated from schemas.
- **Spec refs**: FR-012
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The skill SHALL produce one section per API endpoint with method, path, and description (FR-012.1, FR-012.2, FR-012.3)
  - [x] The skill SHALL include request parameters and body from contract files (FR-012.4)
  - [x] The skill SHALL include response schema from contract files (FR-012.5)
  - [x] The skill SHALL include error codes and meanings from the error catalog contract (FR-012.6)
  - [x] The skill SHALL include authentication requirements (FR-012.7)
  - [x] The skill SHALL include example request/response generated from schemas (FR-012.8)
  - [x] BDD: Given api-contracts.ts defines POST /users with CreateUserInput { email, name, role }, When doc-api-reference runs, Then api-reference.md shows POST /users with fields email, name, role (Section 11.2 Scenario 2)
- **Test requirements**: BDD
- **Depends on**: T32-04
- **Implementation Guidance**:
  - Reference: US-02 acceptance scenario for the contract-to-docs mapping pattern
  - Pattern: Read contract files (`api-contracts.<ext>`, `error-catalog.<ext>`) from the contracts directory; parse type definitions and generate doc sections
  - Known pitfalls: Contract files may use different languages (TypeScript, Python, Go) -- the skill must handle the target language's syntax
  - Files to modify: `.github/skills/doc-api-reference/SKILL.md`

### T32-06 - Write contract-based accuracy rules

- **Description**: Write the skill instructions enforcing that API docs are generated from contract files, NOT from prose interpretation. If contracts exist, they are the source of truth. If no contract files exist for the WP, the skill SHALL skip API doc generation and log that no contracts were found.
- **Spec refs**: FR-013
- **Parallel**: No
- **Acceptance criteria**:
  - [x] API docs SHALL be generated from contract files, NOT from prose interpretation; if contracts exist, they are the source of truth (FR-013)
  - [x] If no contract files exist for the WP, the skill SHALL skip API doc generation and log that no contracts were found (FR-013 error)
  - [x] The skill explicitly states contract files as primary source, spec prose as fallback-only context
- **Test requirements**: BDD
- **Depends on**: T32-05
- **Implementation Guidance**:
  - Pattern: Check for contract files first; if found, generate from contracts; if absent, log and skip
  - Known pitfalls: Do not fall back to generating API docs from spec prose -- this violates the contract-based accuracy requirement
  - Files to modify: `.github/skills/doc-api-reference/SKILL.md`

### T32-07 - Integration verification with coordinator

- **Description**: Verify both skills integrate correctly with the Docs Agent coordinator. Confirm the coordinator discovers doc-architecture and doc-api-reference, dispatches them in canonical order (positions 1 and 2), and passes the correct context.
- **Spec refs**: FR-003, FR-004
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Coordinator discovers both skills via `doc-*/SKILL.md` glob
  - [x] doc-architecture is dispatched before doc-api-reference (canonical order position 1 vs 2)
  - [x] Both skills receive all 6 context items defined in FR-005
  - [x] BDD: Given WP03 adds 2 new API endpoints, When the Docs Agent runs, Then architecture.md is updated And api-reference.md has 2 new endpoint sections (Section 11.2 Scenario 1)
- **Test requirements**: BDD
- **Depends on**: T32-01, T32-02, T32-03, T32-04, T32-05, T32-06
- **Implementation Guidance**:
  - Pattern: Manual invocation of the Docs Agent coordinator with a sample approved WP
  - Known pitfalls: Verify skills read existing docs before writing to test incremental update behavior

## Implementation Notes

Both skills produce markdown content for `.sdd/docs/`. The doc-api-reference skill is unique in that it reads language-specific contract files -- it must handle TypeScript, Python, Go, and Rust contract syntaxes based on the project's target language.

The incremental update pattern (FR-011) is critical: both skills must read existing doc content before writing. Naive file overwrites would destroy documentation from prior WPs.

All implementation artifacts are markdown SKILL.md files. "Testing" means manually invoking each skill with a sample WP and verifying the output docs.

## Risks & Mitigations

- **Risk**: Contract file formats vary by language, making API doc generation complex. **Mitigation**: The skill instructions should specify how to read common contract patterns (TypeScript interfaces, Python dataclasses, Go structs).
- **Risk**: Incremental updates may produce inconsistent section ordering. **Mitigation**: Define a canonical section order in the architecture.md template; the skill appends/updates within that structure.

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-06T12:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-06T12:10:00Z - coder - T32-01/02/03 - completed - doc-architecture SKILL.md fully implemented with frontmatter, input/output contract, 6 generation sections, and incremental update protocol
- 2026-04-06T12:20:00Z - coder - T32-04/05/06 - completed - doc-api-reference SKILL.md fully implemented with contract discovery, multi-language parsing, 6 endpoint doc sections, accuracy rules, and incremental updates
- 2026-04-06T12:25:00Z - coder - T32-07 - completed - Integration verified: coordinator discovers both skills via glob, canonical order correct (pos 1 and 2), all 6 FR-005 context items present
- 2026-04-06T12:25:00Z - coder - lane=for_review - All tasks complete, all acceptance criteria met
- 2026-04-06T13:00:00Z - review-coordinator - lane=done - Verdict: Approved

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-06T13:00:00Z
> **Verdict**: Approved
> **Skills dispatched**: review-spec (PASS), review-security (PASS), review-quality (PASS), review-tests (PASS), review-architecture (PASS), review-performance (PASS), review-docs (PASS), review-deps (PASS)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All 7 tasks have acceptance criteria checked
- [PASS] Activity Log: Correct lane transitions (planned -> doing -> for_review)
- [PASS] Commit granularity: 2 logical commits -- one per skill file grouping related tasks
- [PASS] Encoding: No violations found

### Review Feedback

No feedback items -- all checks passed.

### Warnings

No warnings.

### Cross-Correlation Notes

No cross-correlation findings.

### Statistics

| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 4 | 0 | 0 |
| review-spec | 4 | 0 | 0 |
| review-quality | 4 | 0 | 0 |
| review-security | 0 | 0 | 0 |
| review-tests | 0 | 0 | 0 |
| review-architecture | 1 | 0 | 0 |
| review-performance | 0 | 0 | 0 |
| review-docs | 0 | 0 | 0 |
| review-deps | 0 | 0 | 0 |
| **Total** | **13** | **0** | **0** |
