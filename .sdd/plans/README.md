# Plan Index

> **Generated**: 2026-04-04
> **Updated**: 2026-04-05

---

## Spec 001 - Reviewer V2 Skill-Based Architecture

> **Spec**: `.sdd/specs/001-reviewer-v2-skill-based-architecture.spec.md`

## Work Packages

| ID | Title | Priority | Status | Depends On | Parallelisable |
|----|-------|----------|--------|------------|----------------|
| [WP01](WP01-foundation-scaffolding.md) | Foundation & Scaffolding | P0 | Complete | none | - |
| [WP02](WP02-review-coordinator.md) | Review Coordinator Agent | P1 | Complete | WP01 | No |
| [WP03](WP03-review-spec.md) | Spec Adherence Review Skill | P1 | Complete | WP02 | Yes |
| [WP04](WP04-review-security.md) | Security Review Skill | P1 | Complete | WP02 | Yes |
| [WP05](WP05-review-quality.md) | Code Quality Review Skill | P1 | Complete | WP02 | Yes |
| [WP06](WP06-p2-skills.md) | P2 Skills (tests + architecture) | P2 | Done | WP02 | Yes |
| [WP07](WP07-p3-skills.md) | P3 Skills (perf + docs + deps) | P3 | Done | WP02 | Yes |

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

---

## Spec 002 -- Spec Architect V2

> **Spec**: `.sdd/specs/002-spec-architect-v2.spec.md`

### Work Packages

| ID | Title | Priority | Status | Depends On | Parallelisable |
|----|-------|----------|--------|------------|----------------|
| [WP08](WP08-foundation-spec-architect.md) | Foundation & Skill Directories | P0 | Complete | none | - |
| [WP09](WP09-spec-architect-coordinator.md) | Spec Architect Coordinator | P1 | Complete | WP08 | No |
| [WP10](WP10-requirements-user-stories-skills.md) | Requirements & User Stories Skills | P1 | Complete | WP08, WP09 | Yes |
| [WP11](WP11-data-model-api-design-skills.md) | Data Model & API Design Skills | P1 | Complete | WP08, WP09 | Yes |
| [WP12](WP12-architecture-security-skills.md) | Architecture & Security Skills | P1 | Not Started | WP08, WP09 | Yes |
| [WP13](WP13-test-traceability-skills.md) | Test Strategy & Traceability Skills | P2 | Not Started | WP08, WP09 | Yes |

### MVP Scope

The following work packages constitute the minimum releasable increment: **WP08, WP09, WP10, WP11, WP12**.

- WP08 (P0) creates the directory scaffolding and stub skill files for all 8 spec skills
- WP09 (P1) rewrites the Spec Architect coordinator from monolithic to skill-based dispatch
- WP10-WP12 (P1) implement the 6 core spec skills (requirements, user stories, data model, API design, architecture, security)

WP13 (P2) adds test strategy and traceability skills. These are valuable for spec quality but the coordinator can produce specs without them.

### Dependency & Execution Summary

- **Sequence**: WP08 -> WP09 -> {WP10, WP11, WP12, WP13}
- **Parallelization**: WP10, WP11, WP12, WP13 can all run in parallel after WP09 completes. Each creates independent skill files.
- **Critical path**: WP08 -> WP09 -> WP12 (longest content, 10 tasks)

### Sequencing Notes

WP08 creates the directory structure and stub SKILL.md files that the coordinator's dynamic discovery depends on (FR-009). Without the directories, the coordinator halts with "no spec skills installed."

WP09 is the critical bottleneck: it rewrites the entire spec-architect.agent.md from V1 monolithic to V2 coordinator pattern. All skill WPs (WP10-WP13) depend on the coordinator being in place to dispatch them.

After WP09 completes, WP10-WP13 are fully parallelizable because each creates independent skill files (`.github/skills/spec-*/SKILL.md`). However, WP10 (requirements + user stories) is recommended first because it establishes the reference pattern that other skills follow.

All implementation artifacts are markdown files (.agent.md, SKILL.md). There is no executable code, build system, or test framework. "Testing" means manually invoking the coordinator against a brief and verifying output.

### Task Index

| Task ID | Summary | Work Package | Parallel? |
|---------|---------|--------------|-----------|
| T08-01 | Create spec skill directory structure | WP08 | Yes |
| T08-02 | Create stub SKILL.md files | WP08 | Yes |
| T08-03 | Create spec-patterns.md placeholder | WP08 | Yes |
| T08-04 | Document artifact directory convention | WP08 | Yes |
| T08-05 | Define common skill contract template | WP08 | Yes |
| T09-01 | Refactor agent file YAML frontmatter | WP09 | No |
| T09-02 | Write brief selection logic | WP09 | No |
| T09-03 | Write research phase instructions | WP09 | No |
| T09-04 | Write gap analysis flow | WP09 | No |
| T09-05 | Write accumulator initialization | WP09 | No |
| T09-06 | Write dynamic skill discovery | WP09 | No |
| T09-07 | Write skill dispatch loop | WP09 | No |
| T09-08 | Write companion artifact management | WP09 | Yes |
| T09-09 | Write post-completion validation | WP09 | No |
| T09-10 | Write patterns consumption | WP09 | Yes |
| T09-11 | Write presentation, approval, and commit | WP09 | No |
| T10-01 | Implement spec-requirements SKILL.md | WP10 | No |
| T10-02 | Add Implementation Contract subsections | WP10 | No |
| T10-03 | Add common skill contract compliance | WP10 | No |
| T10-04 | Implement spec-user-stories SKILL.md | WP10 | No |
| T10-05 | Add contract compliance to user-stories | WP10 | Yes |
| T10-06 | Test both skills with sample brief | WP10 | No |
| T11-01 | Implement spec-data-model SKILL.md | WP11 | No |
| T11-02 | Add companion artifact generation (data) | WP11 | No |
| T11-03 | Add contract compliance to data-model | WP11 | Yes |
| T11-04 | Implement spec-api-design SKILL.md | WP11 | No |
| T11-05 | Add companion artifact generation (API) | WP11 | No |
| T11-06 | Add cross-reference validation (API-data) | WP11 | Yes |
| T11-07 | Add contract compliance to api-design | WP11 | Yes |
| T11-08 | Test both skills with partial accumulator | WP11 | No |
| T12-01 | Implement spec-architecture SKILL.md | WP12 | No |
| T12-02 | Add config-schema companion artifact | WP12 | No |
| T12-03 | Add directory structure validation | WP12 | Yes |
| T12-04 | Add virtual environment requirement | WP12 | Yes |
| T12-05 | Add contract compliance to architecture | WP12 | Yes |
| T12-06 | Implement spec-security SKILL.md | WP12 | No |
| T12-07 | Add cross-reference for security | WP12 | Yes |
| T12-08 | Add web research requirement (OWASP) | WP12 | Yes |
| T12-09 | Add contract compliance to security | WP12 | Yes |
| T12-10 | Test both skills with partial accumulator | WP12 | No |
| T13-01 | Implement spec-test-strategy SKILL.md | WP13 | No |
| T13-02 | Add 1:1 BDD scenario mapping | WP13 | No |
| T13-03 | Add BDD/TDD emphasis | WP13 | Yes |
| T13-04 | Add contract compliance to test-strategy | WP13 | Yes |
| T13-05 | Implement spec-traceability SKILL.md | WP13 | No |
| T13-06 | Add traceability matrix validation | WP13 | No |
| T13-07 | Add orphan FR/US detection | WP13 | Yes |
| T13-08 | Add contract compliance to traceability | WP13 | Yes |
| T13-09 | Test both skills with full accumulator | WP13 | No |

**Total**: 6 work packages, 49 tasks

### FR Traceability

Every FR from Spec 002 is assigned to exactly one task:

| FR Range | Assignment | WP |
|----------|------------|-----|
| FR-001 to FR-002 | T09-02 | WP09 |
| FR-003 to FR-004 | T09-03 | WP09 |
| FR-005 to FR-006 | T09-04 | WP09 |
| FR-007 to FR-008 | T09-05 | WP09 |
| FR-009 to FR-010 | T09-06 | WP09 |
| FR-011 to FR-013 | T09-07 | WP09 |
| FR-014 to FR-016 | T09-08 | WP09 |
| FR-017 to FR-018 | T09-09 | WP09 |
| FR-019 | T09-10 | WP09 |
| FR-020 to FR-022 | T09-11 | WP09 |
| FR-023 to FR-028 | T10-03, T10-05, T11-03, T11-07, T12-05, T12-09, T13-04, T13-08 (common contract per skill) | WP10-13 |
| FR-029 to FR-031 | T10-01 | WP10 |
| FR-032 | T10-02 | WP10 |
| FR-033 to FR-035 | T10-04 | WP10 |
| FR-036 | T11-01 | WP11 |
| FR-037 to FR-038 | T11-02 | WP11 |
| FR-039 | T11-04 | WP11 |
| FR-040 to FR-041 | T11-05 | WP11 |
| FR-042 | T11-06 | WP11 |
| FR-043 | T12-01 | WP12 |
| FR-044 | T12-02 | WP12 |
| FR-045 | T12-03 | WP12 |
| FR-046 | T12-04 | WP12 |
| FR-047 | T12-06 | WP12 |
| FR-048 | T12-07 | WP12 |
| FR-049 | T12-08 | WP12 |
| FR-050 | T13-01 | WP13 |
| FR-051 | T13-02 | WP13 |
| FR-052 | T13-03 | WP13 |
| FR-053 | T13-05 | WP13 |
| FR-054 | T13-06 | WP13 |
| FR-055 | T13-07 | WP13 |

### Consistency Notes

Cross-WP consistency audit performed. No inconsistencies found:

- **Data contracts**: All WPs reference identical accumulator file format (Section 7.1). Skills read/write to the same accumulator path.
- **Skill contract**: Common input/output contract (FR-023-028) is implemented identically in each skill WP. Coordinator dispatch prompt template (Section 8.2) matches skill input expectations.
- **Dependency graph**: Verified no circular dependencies. WP08 -> WP09 -> {WP10 || WP11 || WP12 || WP13}. All `Depends on` declarations are valid.
- **Configuration**: Glob patterns (`spec-*/SKILL.md`), file paths, directory names, and artifact naming conventions are consistent across coordinator (WP09) and all skill WPs (WP10-WP13).
- **Test consistency**: All WPs use manual invocation testing. Coverage thresholds (80% code, 90% branch) are consistent in WP13 test strategy.
- **Spec traceability**: All 55 FRs (FR-001 through FR-055) are assigned. No orphan FRs, no duplicate assignments (except FR-023-028 which apply to all skills by design).
