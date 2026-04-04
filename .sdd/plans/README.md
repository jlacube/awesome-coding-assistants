# Plan Index - Reviewer V2 Skill-Based Architecture

> **Spec**: `.sdd/specs/001-reviewer-v2-skill-based-architecture.spec.md`
> **Generated**: 2026-04-04

## Work Packages

| ID | Title | Priority | Status | Depends On | Parallelisable |
|----|-------|----------|--------|------------|----------------|
| [WP01](WP01-foundation-scaffolding.md) | Foundation & Scaffolding | P0 | Complete | none | - |
| [WP02](WP02-review-coordinator.md) | Review Coordinator Agent | P1 | In Progress | WP01 | No |
| [WP03](WP03-review-spec.md) | Spec Adherence Review Skill | P1 | Not Started | WP02 | Yes |
| [WP04](WP04-review-security.md) | Security Review Skill | P1 | Not Started | WP02 | Yes |
| [WP05](WP05-review-quality.md) | Code Quality Review Skill | P1 | Not Started | WP02 | Yes |
| [WP06](WP06-p2-skills.md) | P2 Skills (tests + architecture) | P2 | Not Started | WP02 | Yes |
| [WP07](WP07-p3-skills.md) | P3 Skills (perf + docs + deps) | P3 | Not Started | WP02 | Yes |

## MVP Scope

The following work packages constitute the minimum releasable increment: **WP01, WP02, WP03, WP04, WP05**.

- WP01 (P0) provides the directory structure and deprecates the old reviewer
- WP02 (P1) provides the coordinator that orchestrates multi-skill reviews
- WP03-05 (P1) provide the three core review dimensions: spec adherence, security, code quality

WP06 and WP07 are post-MVP enhancements that add test quality, architecture adherence, performance, documentation, and dependency review dimensions. The coordinator dynamically discovers skills, so P2/P3 skills integrate automatically once installed.

## Dependency & Execution Summary

- **Sequence**: WP01 -> WP02 -> WP03 -> {WP04, WP05} -> {WP06, WP07}
- **Parallelization**: WP04 + WP05 can run in parallel after WP03. WP06 + WP07 can run in parallel after P1 skills are complete.
- **Critical path**: WP01 -> WP02 -> WP03 -> WP04 or WP05 (whichever finishes last) = MVP complete

## Sequencing Notes

WP01 is pure scaffolding (directories, template files, old reviewer deprecation) with no review logic. It must be done first to establish the file structure.

WP02 is the largest and most critical WP: it creates the Review Coordinator agent file containing all orchestration logic (25 FRs). It depends on WP01's directory structure and Orchestrator reference update.

WP03 (review-spec) should be implemented first among the P1 skills because it establishes the reference pattern for all subsequent skills. WP04 and WP05 follow the same contract and can be worked in parallel once the pattern is established.

WP06 and WP07 technically depend only on WP02 (the coordinator), not on the P1 skills. However, their integration verification tasks (T06-07, T07-07) need all earlier skills installed to verify full-suite dispatch. Practically, implement WP06 after P1 skills are complete, and WP07 after WP06.

All implementation artifacts are markdown files (.agent.md, SKILL.md). There is no executable code, build system, or test framework. "Testing" means manually invoking the coordinator against a WP with known issues and verifying the output matches BDD scenarios from the spec.

## Task Index

| Task ID | Summary | Work Package | Parallel? |
|---------|---------|--------------|-----------|
| T01-01 | Create review artifacts directory | WP01 | Yes |
| T01-02 | Create P1 skill directory structure | WP01 | Yes |
| T01-03 | Create initial review-patterns.md template | WP01 | Yes |
| T01-04 | Deprecate old reviewer.agent.md | WP01 | Yes |
| T01-05 | Update Orchestrator agent reference | WP01 | No |
| T01-06 | Verify directory structure and commit | WP01 | No |
| T02-01 | Create coordinator agent file with YAML frontmatter | WP02 | No |
| T02-02 | Write scope selection and artifact chain loading | WP02 | No |
| T02-03 | Write dynamic skill discovery and dispatch ordering | WP02 | No |
| T02-04 | Write coordinator-owned checks (process + encoding) | WP02 | No |
| T02-05 | Write skill dispatch via runSubagent | WP02 | No |
| T02-06 | Write findings aggregation and cross-correlation | WP02 | No |
| T02-07 | Write verdict determination and review report | WP02 | No |
| T02-08 | Write WP lifecycle and Activity Log management | WP02 | No |
| T02-09 | Write patterns file curation | WP02 | No |
| T02-10 | Write re-review, stalled cycle, commit, boundaries | WP02 | No |
| T03-01 | Create review-spec SKILL.md with frontmatter | WP03 | No |
| T03-02 | Write FR classification and adherence checklist | WP03 | No |
| T03-03 | Write stub detection rules | WP03 | Yes |
| T03-04 | Write success criteria verification rules | WP03 | Yes |
| T03-05 | Write severity guidance and N/A handling | WP03 | No |
| T03-06 | Write output format instructions | WP03 | No |
| T04-01 | Create review-security SKILL.md with frontmatter | WP04 | No |
| T04-02 | Write OWASP categories 1-7 checklist | WP04 | Yes |
| T04-03 | Write OWASP categories 8-14 checklist | WP04 | Yes |
| T04-04 | Write spec security cross-reference instructions | WP04 | No |
| T04-05 | Write web research instructions | WP04 | No |
| T04-06 | Write severity guidance, N/A handling, output format | WP04 | No |
| T05-01 | Create review-quality SKILL.md with frontmatter | WP05 | No |
| T05-02 | Write readability, complexity, naming, comment checklist | WP05 | Yes |
| T05-03 | Write error handling, style, dead code, duplication checklist | WP05 | Yes |
| T05-04 | Write severity guidance | WP05 | No |
| T05-05 | Write output format instructions | WP05 | No |
| T06-01 | Create review-tests SKILL.md with frontmatter | WP06 | Yes |
| T06-02 | Write test quality checklist | WP06 | No |
| T06-03 | Write review-tests severity guidance and output format | WP06 | No |
| T06-04 | Create review-architecture SKILL.md with frontmatter | WP06 | Yes |
| T06-05 | Write architecture adherence checklist | WP06 | No |
| T06-06 | Write review-architecture severity guidance and output format | WP06 | No |
| T06-07 | Integration verification of P2 skills with coordinator | WP06 | No |
| T07-01 | Create review-performance SKILL.md with frontmatter | WP07 | Yes |
| T07-02 | Write performance checklist, severity, output format | WP07 | No |
| T07-03 | Create review-docs SKILL.md with frontmatter | WP07 | Yes |
| T07-04 | Write documentation checklist, severity, output format | WP07 | No |
| T07-05 | Create review-deps SKILL.md with frontmatter | WP07 | Yes |
| T07-06 | Write dependencies checklist, severity, output format | WP07 | No |
| T07-07 | Integration verification of all 8 skills with coordinator | WP07 | No |

**Total**: 7 work packages, 46 tasks

## FR Traceability

Every FR from the spec is assigned to exactly one task:

| FR Range | Assignment | WP |
|----------|------------|-----|
| FR-001 to FR-002 | T02-02 | WP02 |
| FR-003 to FR-004 | T02-03 | WP02 |
| FR-005 to FR-006 | T02-04 | WP02 |
| FR-007 to FR-009 | T02-05 | WP02 |
| FR-010 to FR-011 | T02-06 | WP02 |
| FR-012 to FR-014 | T02-07 | WP02 |
| FR-015 to FR-017 | T02-08 | WP02 |
| FR-018 to FR-019 | T02-09 | WP02 |
| FR-020 to FR-024 | T02-10 | WP02 |
| FR-025 to FR-029 | T03-01/06, T04-01/06, T05-01/05, WP06, WP07 (common contract per skill) | WP03-07 |
| FR-030 to FR-031 | T03-02 | WP03 |
| FR-032 | T03-03 | WP03 |
| FR-033 | T03-04 | WP03 |
| FR-034 | T04-02, T04-03 | WP04 |
| FR-035 | T04-04 | WP04 |
| FR-036 | T04-05 | WP04 |
| FR-037 | T05-02, T05-03 | WP05 |
| FR-038 | T05-04 | WP05 |
| FR-039 | T05-04 | WP05 |
| FR-040 | T06-02 | WP06 |
| FR-041 | T06-03 | WP06 |
| FR-042 | T06-05 | WP06 |
| FR-043 | T06-06 | WP06 |
| FR-044 to FR-045 | T07-02 | WP07 |
| FR-046 to FR-047 | T07-04 | WP07 |
| FR-048 to FR-049 | T07-06 | WP07 |
| FR-050 | T02-07 | WP02 |

## Consistency Notes

Cross-WP consistency audit performed before plan submission. No inconsistencies found:

- **Data contracts**: All WPs reference identical findings file format (Section 7.1). Finding prefixes are globally unique across all skills.
- **Skill contract**: Common input/output contract (FR-025-029) is implemented identically in each skill WP. Coordinator dispatch prompt (Section 8.3) matches skill input expectations.
- **Dependency graph**: Verified no circular dependencies. All `Depends on` declarations are valid.
- **Configuration**: Glob patterns, file paths, and directory names are consistent across coordinator (WP02) and all skill WPs.
- **Sequencing note**: WP03 is recommended before WP04/WP05 to establish the reference skill pattern, though technically all three can run in parallel (they share no code dependencies). Integration verification tasks in WP06/WP07 require earlier skills to be installed.
