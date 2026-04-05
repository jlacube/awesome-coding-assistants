# Plan Index

> **Generated**: 2026-04-04
> **Updated**: 2026-04-06T00:00:00Z

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
| [WP12](WP12-architecture-security-skills.md) | Architecture & Security Skills | P1 | Complete | WP08, WP09 | Yes |
| [WP13](WP13-test-traceability-skills.md) | Test Strategy & Traceability Skills | P2 | Complete | WP08, WP09 | Yes |

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

---

## Spec 003 -- Planner V2

> **Spec**: `.sdd/specs/003-planner-v2.spec.md`

### Work Packages

| ID | Title | Priority | Status | Depends On | Parallelisable |
|----|-------|----------|--------|------------|----------------|
| [WP14](WP14-foundation-plan-skills.md) | Foundation & Plan Skill Scaffolding | P0 | Complete | none | - |
| [WP15](WP15-planner-coordinator.md) | Planner Coordinator Rewrite | P1 | Complete | WP14 | No |
| [WP16](WP16-phase1-decomposition-acceptance.md) | Phase 1: Decomposition & Acceptance Skills | P1 | Complete | WP14, WP15 | No |
| [WP17](WP17-phase2-interface-data-skills.md) | Phase 2: Interface Contracts & Data Schemas Skills | P1 | Complete | WP14, WP15 | Yes |
| [WP18](WP18-phase2-api-state-error-skills.md) | Phase 2: API Contracts, State Machines & Error Catalogs Skills | P1 | Complete | WP14, WP15 | Yes |
| [WP19](WP19-phase2-cross-wp-validation.md) | Phase 2: Cross-WP Validation Skill | P1 | Complete | WP14, WP15, WP16, WP17, WP18 | No |

### MVP Scope

The following work packages constitute the minimum releasable increment: **WP14, WP15, WP16**.

- WP14 (P0) creates the directory scaffolding, stub skill files for all 8 plan skills, and the common contract (PLAN-SKILL-CONTRACT.md)
- WP15 (P1) rewrites the Planner coordinator from monolithic to skill-based dispatch with auto-loop gap resolution
- WP16 (P1) implements Phase 1 skills (plan-decomposition + plan-acceptance) that produce the core WP decomposition

WP17-WP19 are post-MVP enhancements that add Phase 2 contract generation (interfaces, data schemas, API contracts, state machines, error catalogs, cross-WP validation). The coordinator dispatches Phase 2 skills only when they are installed, so the planner works without them.

### Dependency & Execution Summary

- **Sequence**: WP14 -> WP15 -> WP16 -> {WP17, WP18} -> WP19
- **Parallelization**: WP17 and WP18 can run in parallel after WP15 completes. WP19 must run after WP17 + WP18 (it validates their outputs).
- **Critical path**: WP14 -> WP15 -> WP16 -> WP17 or WP18 (whichever finishes last) -> WP19

### Sequencing Notes

WP14 creates the directory structure and stub SKILL.md files that the coordinator's dynamic discovery depends on (FR-009). Without the directories, the coordinator cannot discover plan skills.

WP15 is the critical bottleneck: it rewrites the entire planner.agent.md from V1 monolithic to V2 skill-based coordinator with two-phase dispatch, auto-loop gap resolution, and skill manifest support. All skill WPs depend on the coordinator being in place.

WP16 (Phase 1: plan-decomposition + plan-acceptance) must be implemented before Phase 2 skills because Phase 2 skills read the plan accumulator produced by Phase 1. WP16 also establishes the reference pattern for all subsequent plan skills.

WP17 and WP18 (Phase 2 contract generation skills) are fully parallelizable because each creates independent skill files. WP17 covers interface contracts and data schemas; WP18 covers API contracts, state machines, and error catalogs.

WP19 (cross-WP validation) must run last because it audits ALL preceding skill outputs for consistency.

All implementation artifacts are markdown files (.agent.md, SKILL.md). There is no executable code, build system, or test framework. "Testing" means manually invoking the coordinator against a validated spec and verifying output.

### Task Index

| Task ID | Summary | Work Package | Parallel? |
|---------|---------|--------------|-----------|
| T14-01 | Create plan skill directory structure | WP14 | Yes |
| T14-02 | Create stub SKILL.md files for all 8 plan skills | WP14 | Yes |
| T14-03 | Create PLAN-SKILL-CONTRACT.md common contract | WP14 | No |
| T14-04 | Define contracts directory structure | WP14 | Yes |
| T14-05 | Create plan-patterns.md placeholder | WP14 | Yes |
| T14-06 | Define manifest header format | WP14 | No |
| T14-07 | Verify encoding compliance | WP14 | No |
| T15-01 | Refactor planner.agent.md YAML frontmatter | WP15 | No |
| T15-02 | Write spec selection and confirmation logic | WP15 | No |
| T15-03 | Write spec validation (status check) | WP15 | No |
| T15-04 | Write companion artifact loading | WP15 | No |
| T15-05 | Write spec completeness pre-check | WP15 | No |
| T15-06 | Write auto-loop to Spec Architect | WP15 | No |
| T15-07 | Write research phase instructions | WP15 | No |
| T15-08 | Write dynamic skill discovery | WP15 | No |
| T15-09 | Write two-phase skill dispatch | WP15 | No |
| T15-10 | Write plan presentation and human gate | WP15 | No |
| T15-11 | Write commit and handoff instructions | WP15 | No |
| T15-12 | Verify encoding compliance | WP15 | No |
| T16-01 | Implement plan-decomposition SKILL.md structure | WP16 | No |
| T16-02 | Implement WP identification logic | WP16 | No |
| T16-03 | Implement task decomposition logic | WP16 | No |
| T16-04 | Implement WP file generation | WP16 | No |
| T16-05 | Implement README index generation | WP16 | No |
| T16-06 | Implement plan-acceptance SKILL.md structure | WP16 | No |
| T16-07 | Implement acceptance criteria extraction | WP16 | No |
| T16-08 | Implement BDD scenario mapping | WP16 | No |
| T16-09 | Verify encoding compliance | WP16 | No |
| T17-01 | Implement plan-interface-contracts SKILL.md | WP17 | No |
| T17-02 | Implement interface contract generation | WP17 | No |
| T17-03 | Implement shared interface deduplication | WP17 | No |
| T17-04 | Implement plan-data-schemas SKILL.md | WP17 | Yes |
| T17-05 | Implement data schema generation | WP17 | No |
| T17-06 | Implement shared entity deduplication | WP17 | No |
| T17-07 | Implement manifest headers and 800-line compliance | WP17 | No |
| T17-08 | Verify encoding compliance | WP17 | No |
| T18-01 | Implement plan-api-contracts SKILL.md structure | WP18 | No |
| T18-02 | Implement API contract generation logic | WP18 | No |
| T18-03 | Implement error response types per endpoint | WP18 | No |
| T18-04 | Implement plan-state-machines SKILL.md | WP18 | Yes |
| T18-05 | Implement plan-error-catalogs SKILL.md | WP18 | Yes |
| T18-06 | Implement manifest headers and 800-line compliance | WP18 | No |
| T18-07 | Verify encoding compliance | WP18 | No |
| T19-01 | Implement plan-cross-wp-validation SKILL.md structure | WP19 | No |
| T19-02 | Implement data contract consistency check | WP19 | Yes |
| T19-03 | Implement API/interface contract consistency check | WP19 | Yes |
| T19-04 | Implement dependency integrity check | WP19 | Yes |
| T19-05 | Implement configuration consistency and config schema generation | WP19 | Yes |
| T19-06 | Implement test consistency and spec traceability checks | WP19 | Yes |
| T19-07 | Implement contract-to-task alignment check | WP19 | Yes |
| T19-08 | Implement inconsistency fix and documentation | WP19 | No |
| T19-09 | Implement 100% spec artifact coverage verification | WP19 | No |
| T19-10 | Verify encoding compliance | WP19 | No |

**Total**: 6 work packages, 54 tasks

### FR Traceability

Every FR from Spec 003 is assigned to exactly one task:

| FR Range | Assignment | WP |
|----------|------------|-----|
| FR-001 | T15-02 | WP15 |
| FR-002 | T15-04 | WP15 |
| FR-003 | T15-03 | WP15 |
| FR-004 | T15-05 | WP15 |
| FR-005 | T15-04 | WP15 |
| FR-006 | T15-06 | WP15 |
| FR-007 | T15-07 | WP15 |
| FR-008 | T15-07 | WP15 |
| FR-009 | T15-08 | WP15 |
| FR-010 | T15-08 | WP15 |
| FR-011 | T15-09 | WP15 |
| FR-012 | T15-09 | WP15 |
| FR-013 | T15-09, T17-07, T18-06 | WP15, WP17, WP18 |
| FR-014 | T15-09 | WP15 |
| FR-015 | T15-09 | WP15 |
| FR-016 | T15-09 | WP15 |
| FR-017 | T15-10 | WP15 |
| FR-018 | T15-10 | WP15 |
| FR-019 | T15-11 | WP15 |
| FR-020 | T15-11 | WP15 |
| FR-021 | T15-11 | WP15 |
| FR-022 | T15-11 | WP15 |
| FR-023 | T14-03, all skill WPs (common contract) | WP14, WP16-19 |
| FR-024 | T14-03, all skill WPs (common contract) | WP14, WP16-19 |
| FR-025 | T14-01 | WP14 |
| FR-026 | T14-02 | WP14 |
| FR-027 | T14-04 | WP14 |
| FR-028 | T16-01 | WP16 |
| FR-029 | T16-02 | WP16 |
| FR-030 | T16-03 | WP16 |
| FR-031 | T16-03 | WP16 |
| FR-032 | T16-04 | WP16 |
| FR-033 | T16-05 | WP16 |
| FR-034 | T16-06 | WP16 |
| FR-035 | T16-07 | WP16 |
| FR-036 | T16-08 | WP16 |
| FR-037 | T17-01 | WP17 |
| FR-038 | T17-02 | WP17 |
| FR-039 | T14-06, T17-07, T18-06 | WP14, WP17, WP18 |
| FR-040 | T17-03 | WP17 |
| FR-041 | T17-04, T17-05 | WP17 |
| FR-042 | T17-06 | WP17 |
| FR-043 | T18-01, T18-02 | WP18 |
| FR-044 | T18-02 | WP18 |
| FR-045 | T18-03 | WP18 |
| FR-046 | T18-04 | WP18 |
| FR-047 | T18-04 | WP18 |
| FR-048 | T18-05 | WP18 |
| FR-049 | T18-05 | WP18 |
| FR-050 | T18-05 | WP18 |
| FR-051 | T19-02 through T19-07 | WP19 |
| FR-052 | T19-05 | WP19 |
| FR-053 | T19-08 | WP19 |
| FR-054 | T19-09 | WP19 |

### Consistency Notes

Cross-WP consistency audit performed before plan submission. Findings:

- **Data contracts**: All WPs reference the same plan accumulator format. Contract output paths follow consistent pattern: `.sdd/plans/contracts/<WP-slug>/<artifact-type>.<ext>`. Shared contracts go to `.sdd/plans/contracts/shared/`.
- **Skill contract**: Common plan-skill contract (FR-023, FR-024) defined in WP14 (PLAN-SKILL-CONTRACT.md) and referenced identically in all skill WPs (WP16-WP19).
- **Dependency graph**: No circular dependencies. WP14 -> WP15 -> WP16 -> {WP17 || WP18} -> WP19. All `Depends on` declarations verified valid.
- **Configuration**: Glob patterns (`plan-*/SKILL.md`), file paths, directory names, manifest header format, and 800-line block limit are consistent across coordinator (WP15) and all skill WPs.
- **Test consistency**: All WPs use manual invocation testing. Coverage thresholds (80% code, 90% branch) are referenced consistently in WP16 (plan-acceptance) and WP19 (cross-validation).
- **Spec traceability**: All 54 FRs (FR-001 through FR-054) are assigned. FR-013 (800-line blocks), FR-023/FR-024 (common contract), and FR-039 (manifest headers) are shared across multiple WPs by design.
- **Phase ordering**: Phase 1 skills (WP16) populate the plan accumulator that Phase 2 skills (WP17-WP19) consume. This ordering is enforced by the coordinator's two-phase dispatch (FR-011, FR-012).

---

## Spec 004 -- Coder V2

> **Spec**: `.sdd/specs/004-coder-v2.spec.md`

### Work Packages

| ID | Title | Priority | Status | Depends On | Parallelisable |
|----|-------|----------|--------|------------|----------------|
| [WP20](WP20-foundation-coder-skills.md) | Foundation: Coder Skill Scaffolding | P0 | Complete | none | - |
| [WP21](WP21-coder-coordinator.md) | Coder Coordinator | P1 | Done | WP20 | No |
| [WP22](WP22-env-setup-implementation-skills.md) | Environment Setup & Core Implementation Skills | P1 | Not Started | WP20, WP21 | Yes |
| [WP23](WP23-test-skills.md) | Test Skills | P1 | Complete | WP20, WP21 | Yes |
| [WP24](WP24-debug-skill.md) | Debug Skill | P1 | Complete | WP20, WP21 | Yes |

### MVP Scope

All work packages are MVP. The Coder V2 requires all 5 components to function:

- WP20 (P0) creates the directory scaffolding, stub skill files, common skill contract, and code-patterns.md
- WP21 (P1) rewrites the Coder coordinator from monolithic to skill-based dispatch with debug retry logic
- WP22 (P1) implements the environment setup and core implementation skills (code-env-setup, code-implementation)
- WP23 (P1) implements the testing skills (code-unit-tests, code-integration-tests)
- WP24 (P1) implements the conditional debug skill (code-debug)

### Dependency & Execution Summary

- **Sequence**: WP20 -> WP21 -> {WP22, WP23, WP24}
- **Parallelization**: WP22, WP23, and WP24 can all run in parallel after WP21 completes. Each creates independent SKILL.md files.
- **Critical path**: WP20 -> WP21 -> WP22 (longest content, 8 tasks)

### Sequencing Notes

WP20 creates the directory structure and stub SKILL.md files that the coordinator's dynamic discovery depends on (FR-005). Without the directories and the glob pattern `.github/skills/code-*/SKILL.md`, the coordinator halts with "no coding skills installed."

WP21 is the critical bottleneck: it rewrites coder.agent.md from monolithic single-pass implementation to a coordinator that dispatches 5 sequential coding skills. It contains the most complex state management (debug retry logic, task tracking, coverage verification). All skill WPs depend on the coordinator being in place.

After WP21 completes, WP22-WP24 are fully parallelizable because each creates independent SKILL.md files. However, WP22 (env-setup + implementation) is recommended first because these are the "producing" skills that the testing and debug skills depend on at runtime. WP23 (test skills) is recommended second, and WP24 (debug) last, mirroring the canonical skill dispatch order from FR-006.

All implementation artifacts are markdown files (.agent.md, SKILL.md). There is no executable code, build system, or test framework. "Testing" means manually invoking the Coder coordinator against a WP with contract files and verifying the output matches BDD scenarios from the spec.

### Task Index

| Task ID | Summary | Work Package | Parallel? |
|---------|---------|--------------|-----------|
| T20-01 | Create 5 coding skill directories | WP20 | No |
| T20-02 | Create stub SKILL.md files with YAML frontmatter | WP20 | No |
| T20-03 | Create CODER-SKILL-CONTRACT.md | WP20 | No |
| T20-04 | Create code-patterns.md placeholder | WP20 | Yes |
| T20-05 | Refactor coder.agent.md YAML frontmatter | WP20 | No |
| T20-06 | Verify directory structure and encoding compliance | WP20 | No |
| T21-01 | Write WP selection and user interaction | WP21 | No |
| T21-02 | Write artifact chain loading | WP21 | No |
| T21-03 | Write contract file validation | WP21 | No |
| T21-04 | Write patterns consumption | WP21 | No |
| T21-05 | Write dynamic skill discovery and ordering | WP21 | No |
| T21-06 | Write skill dispatch via runSubagent | WP21 | No |
| T21-07 | Write conditional debug dispatch with retry logic | WP21 | No |
| T21-08 | Write task state tracking and WP lifecycle | WP21 | No |
| T21-09 | Write post-completion handoff and coverage verification | WP21 | No |
| T21-10 | Write commit policy and handoff prompts | WP21 | No |
| T22-01 | Create code-env-setup SKILL.md structure | WP22 | No |
| T22-02 | Write environment detection and creation logic | WP22 | No |
| T22-03 | Write dependency installation and coverage tooling setup | WP22 | No |
| T22-04 | Write baseline verification and failure handling | WP22 | No |
| T22-05 | Create code-implementation SKILL.md structure | WP22 | Yes |
| T22-06 | Write contract-first implementation logic | WP22 | No |
| T22-07 | Write implementation constraints and scope rules | WP22 | No |
| T22-08 | Integration verification of both skills with coordinator | WP22 | No |
| T23-01 | Create code-unit-tests SKILL.md structure | WP23 | No |
| T23-02 | Write unit test generation logic | WP23 | No |
| T23-03 | Write test validity rules | WP23 | Yes |
| T23-04 | Write test execution and coverage threshold enforcement | WP23 | No |
| T23-05 | Create code-integration-tests SKILL.md structure | WP23 | Yes |
| T23-06 | Write integration test generation logic | WP23 | No |
| T23-07 | Write integration test execution and reporting | WP23 | No |
| T23-08 | Integration verification of both test skills with coordinator | WP23 | No |
| T24-01 | Create code-debug SKILL.md structure | WP24 | No |
| T24-02 | Write failure diagnosis logic | WP24 | No |
| T24-03 | Write fix prioritization and safety constraints | WP24 | No |
| T24-04 | Write re-run verification and regression detection | WP24 | No |
| T24-05 | Write escalation reporting format | WP24 | Yes |
| T24-06 | Integration verification with coordinator | WP24 | No |

**Total**: 5 work packages, 38 tasks

### FR Traceability

Every FR from Spec 004 is assigned to exactly one task:

| FR Range | Assignment | WP |
|----------|------------|-----|
| FR-001 | T21-01 | WP21 |
| FR-002 | T21-02 | WP21 |
| FR-003 | T21-03 | WP21 |
| FR-004 | T21-04 | WP21 |
| FR-005 | T20-01, T20-02, T21-05 | WP20, WP21 |
| FR-006 | T21-05 | WP21 |
| FR-007 | T21-06 | WP21 |
| FR-008 | T21-06 | WP21 |
| FR-009 | T21-06 | WP21 |
| FR-010 | T21-07 | WP21 |
| FR-011 | T21-08 | WP21 |
| FR-012 | T21-08 | WP21 |
| FR-013 | T21-08 | WP21 |
| FR-014 | T21-09 | WP21 |
| FR-015 | T21-09 | WP21 |
| FR-016 | T21-10 | WP21 |
| FR-017 | T20-03, T22-01, T22-05, T23-01, T23-05, T24-01 (common contract per skill) | WP20, WP22-24 |
| FR-018 | T20-03, T22-01, T22-05, T23-01, T23-05, T24-01 (common contract per skill) | WP20, WP22-24 |
| FR-019 | T20-03, T22-01, T22-05, T23-01, T23-05, T24-01 (common contract per skill) | WP20, WP22-24 |
| FR-020 | T22-02, T22-03, T22-04 | WP22 |
| FR-021 | T22-04 | WP22 |
| FR-022 | T22-03 | WP22 |
| FR-023 | T22-06 | WP22 |
| FR-024 | T22-06 | WP22 |
| FR-025 | T22-07 | WP22 |
| FR-026 | T22-07 | WP22 |
| FR-027 | T23-02 | WP23 |
| FR-028 | T23-03 | WP23 |
| FR-029 | T23-04 | WP23 |
| FR-030 | T23-04 | WP23 |
| FR-031 | T23-06 | WP23 |
| FR-032 | T23-06 | WP23 |
| FR-033 | T23-07 | WP23 |
| FR-034 | T24-02 | WP24 |
| FR-035 | T24-03 | WP24 |
| FR-036 | T24-03 | WP24 |
| FR-037 | T24-04 | WP24 |

### Consistency Notes

Cross-WP consistency audit performed before plan submission. No inconsistencies found:

- **Skill contract**: Common coder-skill contract (FR-017, FR-018, FR-019) defined in WP20 (CODER-SKILL-CONTRACT.md) and referenced identically in all skill WPs (WP22-WP24). Input contract has 8 fields. Output contract has 6 fields.
- **Dependency graph**: No circular dependencies. WP20 -> WP21 -> {WP22 || WP23 || WP24}. All `Depends on` declarations verified valid.
- **Configuration**: Glob patterns (`code-*/SKILL.md`), file paths, directory names consistent across coordinator (WP21) and all skill WPs (WP22-WP24).
- **Test consistency**: All WPs use manual invocation testing. Coverage thresholds (80% code, 90% branch) are referenced consistently in WP23 (test skills) and WP21 (coordinator coverage verification).
- **Spec traceability**: All 37 FRs (FR-001 through FR-037) are assigned. FR-005 (discovery), FR-017/FR-018/FR-019 (common contract) are shared across multiple WPs by design.
- **Phase ordering**: Skills are dispatched in canonical order: code-env-setup -> code-implementation -> code-unit-tests -> code-integration-tests -> code-debug (conditional). This ordering is enforced by the coordinator (FR-006).
- **Debug coordination**: Test skills (WP23) report test_results in FR-019 format. Coordinator (WP21) reads test_results.fail_count to decide debug dispatch. Debug skill (WP24) receives failing test output and attempt counter. All three WPs use consistent data contracts.

---

## Spec 005 -- Review Spec Completeness & Contract-Aware Review

> **Spec**: `.sdd/specs/005-review-spec-completeness.spec.md`

### Work Packages

| ID | Title | Priority | Status | Depends On | Parallelisable |
|----|-------|----------|--------|------------|----------------|
| [WP25](WP25-review-spec-completeness.md) | review-spec-completeness Skill | P1 | Not Started | none | Yes |
| [WP26](WP26-review-spec-contract-aware.md) | review-spec Contract-Aware Expansion | P1 | Complete | none | Yes |

### MVP Scope

Both work packages are MVP:

- WP25 (P1) creates the new review-spec-completeness skill that validates spec implementation-readiness before planning
- WP26 (P1) expands the existing review-spec skill with contract-aware code review capabilities

Both features deliver SC-001 (pre-planning validation gate) and SC-002 (contract-aware review) respectively. Neither requires Review Coordinator changes (SC-003).

### Dependency & Execution Summary

- **Sequence**: {WP25, WP26} (fully parallel -- no inter-WP dependencies)
- **Parallelization**: WP25 and WP26 modify different files (new SKILL.md vs existing SKILL.md) and can run in parallel
- **Critical path**: WP25 or WP26 (whichever finishes last) = MVP complete

### Sequencing Notes

WP25 creates a brand new skill file at `.github/skills/review-spec-completeness/SKILL.md`. WP26 expands the existing skill at `.github/skills/review-spec/SKILL.md`. Since these are different files with no shared dependencies, both WPs can run in parallel.

No foundation WP is needed because the only scaffolding (creating the `review-spec-completeness/` directory) is trivially included in WP25's first task. No coordinator changes are needed (SC-003).

All implementation artifacts are markdown files (SKILL.md). There is no executable code, build system, or test framework. "Testing" means manually invoking the skill against a spec or implementation with known issues and verifying the findings output matches BDD scenarios.

### Task Index

| Task ID | Summary | Work Package | Parallel? |
|---------|---------|--------------|-----------|
| T25-01 | Create review-spec-completeness directory and SKILL.md with frontmatter | WP25 | No |
| T25-02 | Write finding format and verdict output sections | WP25 | No |
| T25-03 | Write obligation language and ambiguity checks | WP25 | Yes |
| T25-04 | Write error behavior check | WP25 | Yes |
| T25-05 | Write data model completeness check | WP25 | Yes |
| T25-06 | Write API endpoint completeness check | WP25 | Yes |
| T25-07 | Write state machine completeness check | WP25 | Yes |
| T25-08 | Write traceability matrix check | WP25 | Yes |
| T25-09 | Write integration strategy and security requirements checks | WP25 | Yes |
| T25-10 | Write artifact consistency check | WP25 | Yes |
| T25-11 | Integration verification with Review Coordinator | WP25 | No |
| T26-01 | Preserve existing behavior and add contract-aware overview | WP26 | No |
| T26-02 | Write interface contract check with token-level comparison | WP26 | No |
| T26-03 | Write data schema contract check with token-level comparison | WP26 | No |
| T26-04 | Write API, state machine, and error catalog contract checks | WP26 | No |
| T26-05 | Write contract mismatch finding format | WP26 | Yes |
| T26-06 | Write fallback to prose-only handling | WP26 | Yes |
| T26-07 | Integration verification with Review Coordinator | WP26 | No |

**Total**: 2 work packages, 18 tasks

### FR Traceability

Every FR from Spec 005 is assigned to exactly one task:

| FR | Task | Status |
|----|------|--------|
| FR-001 | T25-01 | Covered |
| FR-002 | T25-01 | Covered |
| FR-003 | T25-03 | Covered |
| FR-004 | T25-04 | Covered |
| FR-005 | T25-05 | Covered |
| FR-006 | T25-06 | Covered |
| FR-007 | T25-07 | Covered |
| FR-008 | T25-08 | Covered |
| FR-009 | T25-09 | Covered |
| FR-010 | T25-03 | Covered |
| FR-011 | T25-10 | Covered |
| FR-012 | T25-09 | Covered |
| FR-013 | T25-02 | Covered |
| FR-014 | T25-02 | Covered |
| FR-015 | T26-01 | Covered |
| FR-016 | T26-02, T26-03, T26-04 | Covered |
| FR-017 | T26-02, T26-03, T26-04 | Covered |
| FR-018 | T26-05 | Covered |
| FR-019 | T26-06 | Covered |

**FR coverage**: 19/19 FRs assigned (100%).

### Consistency Notes

Cross-WP consistency audit performed. No inconsistencies found:

- **Data contracts**: Both WPs reference the same finding format structures from Section 7. Completeness findings use SPEC-COMP-XXX prefix; contract findings use SPEC-CONTRACT-XXX prefix. No overlap.
- **Skill contract**: Both skills follow the existing review skill discovery pattern (`review-*/SKILL.md`) and standard findings format. No coordinator changes needed.
- **Dependency graph**: No dependencies between WP25 and WP26. Both modify different files. No circular dependencies.
- **Configuration**: Glob pattern `review-*/SKILL.md` is consistent. Contract file paths `.sdd/plans/contracts/<WP-slug>/` are consistent with Planner V2 output (Spec 003).
- **Test consistency**: Both WPs use manual invocation testing against synthetic fixtures.
- **Spec traceability**: All 19 FRs assigned. FR-016 and FR-017 span multiple tasks by design (FR-016 has 5 sub-checks across 3 tasks; FR-017 is the comparison methodology used by all contract checks).

---

## Spec 006 -- Handoff Schemas & Domain-Specific Patterns

> **Spec**: `.sdd/specs/006-handoff-schemas-patterns.spec.md`

### Work Packages

| ID | Title | Priority | Status | Depends On | Parallelisable |
|----|-------|----------|--------|------------|----------------|
| [WP27](WP27-handoff-schema-definitions.md) | Handoff Schema Definitions | P1 | Complete | none | Yes |
| [WP28](WP28-domain-specific-pattern-files.md) | Domain-Specific Pattern Files & Migration | P1 | Not Started | none | Yes |
| [WP29](WP29-agent-coordinator-integration.md) | Agent Coordinator Integration | P1 | Not Started | WP27, WP28 | No |

### MVP Scope

All 3 work packages are MVP:

- WP27 (P1) creates 8 YAML handoff schema files in `.github/schemas/` defining agent-to-agent contracts
- WP28 (P1) creates/updates 4 domain-specific pattern files in `.sdd/reviews/` and migrates the legacy single patterns file
- WP29 (P1) wires schemas and patterns into agent coordinator files (schema validation, pattern consumption, automated curation)

WP27 and WP28 can run in parallel (they create independent artifact types). WP29 depends on both because it integrates their outputs into agent coordinators.

### Dependency & Execution Summary

- **Sequence**: {WP27, WP28} -> WP29
- **Parallelization**: WP27 and WP28 can run in parallel (schema YAML files vs. pattern Markdown files -- no shared state)
- **Critical path**: WP27 or WP28 (whichever finishes last) -> WP29

### Sequencing Notes

WP27 creates 8 YAML schema files in `.github/schemas/`. These are pure content authoring -- no agent file modifications. The reference schema (spec-to-planner) should be written first as it has the full example from FR-003.

WP28 creates/updates 4 pattern files in `.sdd/reviews/` and migrates the legacy `review-patterns.md`. Two files already exist (spec-patterns.md, code-patterns.md) and need format normalization. Two are new (plan-patterns.md, doc-patterns.md).

WP29 is the integration WP that modifies existing agent `.agent.md` files. Schema validation is added as Step 0 in every coordinator. Pattern consumption is formalized. The Review Coordinator gets the most changes (curation, retirement, commit format).

All implementation artifacts are YAML and Markdown files. There is no executable code, build system, or test framework. "Testing" means manually invoking agents and verifying behavior matches BDD scenarios from the spec.

### Task Index

| Task ID | Summary | Work Package | Parallel? |
|---------|---------|--------------|-----------|
| T27-01 | Create schemas directory and define schema template | WP27 | No |
| T27-02 | Author ideation-to-spec.schema.yaml | WP27 | Yes |
| T27-03 | Author spec-to-planner.schema.yaml | WP27 | Yes |
| T27-04 | Author planner-to-coder.schema.yaml | WP27 | Yes |
| T27-05 | Author coder-to-reviewer.schema.yaml | WP27 | Yes |
| T27-06 | Author rework handoff schemas (3 files) | WP27 | Yes |
| T27-07 | Author orchestrator-handoff.schema.yaml | WP27 | Yes |
| T27-08 | Verify schema compliance and add maintenance docs | WP27 | No |
| T28-01 | Create plan-patterns.md with FR-009 structure | WP28 | Yes |
| T28-02 | Create doc-patterns.md with FR-009 structure | WP28 | Yes |
| T28-03 | Update spec-patterns.md to FR-009 format | WP28 | Yes |
| T28-04 | Update code-patterns.md to FR-009 format | WP28 | Yes |
| T28-05 | Migrate legacy review-patterns.md to domain files | WP28 | No |
| T28-06 | Verify migration idempotency | WP28 | No |
| T28-07 | Verify pattern ID format compliance | WP28 | No |
| T29-01 | Add schema validation to Spec Architect coordinator | WP29 | Yes |
| T29-02 | Add schema validation to Planner coordinator | WP29 | Yes |
| T29-03 | Add schema validation to Coder coordinator | WP29 | Yes |
| T29-04 | Add schema validation to Review Coordinator | WP29 | Yes |
| T29-05 | Formalize domain-specific pattern consumption | WP29 | No |
| T29-06 | Add automated pattern curation to Review Coordinator | WP29 | No |
| T29-07 | Add pattern retirement logic to Review Coordinator | WP29 | No |
| T29-08 | Add pattern curation commit format | WP29 | Yes |
| T29-09 | Verify schema validation blocks invalid handoffs | WP29 | No |
| T29-10 | Verify pattern isolation across domains | WP29 | No |

**Total**: 3 work packages, 25 tasks

### FR Traceability

Every FR from Spec 006 is assigned to exactly one task (or documented multi-task spans):

| FR | Task(s) | Status |
|----|---------|--------|
| FR-001 | T27-02, T27-03, T27-04, T27-05, T27-06, T27-07 | Covered (one schema per sub-item) |
| FR-002 | T27-02, T27-03, T27-04, T27-05, T27-06, T27-07 | Covered (structure applied per schema) |
| FR-003 | T27-01, T27-03 | Covered (template + reference schema) |
| FR-004 | T29-01, T29-02, T29-03, T29-04 | Covered (validation per coordinator) |
| FR-005 | T29-01, T29-02, T29-03, T29-04 | Covered (first-action per coordinator) |
| FR-006 | T27-08 | Covered |
| FR-007 | T27-01, T27-08 | Covered |
| FR-008 | T28-01, T28-02, T28-03, T28-04 | Covered (one task per domain file) |
| FR-009 | T28-01, T28-02, T28-03, T28-04 | Covered (format applied per file) |
| FR-010 | T28-03, T28-04, T28-07 | Covered |
| FR-011 | T29-05 | Covered |
| FR-012 | T29-05 | Covered |
| FR-013 | T29-06 | Covered |
| FR-014 | T29-07 | Covered |
| FR-015 | T29-08 | Covered |
| FR-016 | T28-05, T28-06 | Covered |

**FR coverage**: 16/16 FRs assigned (100%).

### Consistency Notes

Cross-WP consistency audit performed. No inconsistencies found:

- **Data contracts**: WP27 creates schema YAML files referenced by WP29's validation logic. Schema file paths are consistent between WP27 task descriptions and WP29 validation references (e.g., `.github/schemas/spec-to-planner.schema.yaml`).
- **Pattern files**: WP28 creates/updates pattern files referenced by WP29's consumption logic. File paths are consistent (e.g., `.sdd/reviews/plan-patterns.md`).
- **Dependency graph**: No circular dependencies. {WP27 || WP28} -> WP29. All `Depends on` declarations verified valid.
- **Configuration**: Schema directory path `.github/schemas/` and pattern directory path `.sdd/reviews/` are consistent across all 3 WPs. Schema version `handoff/v1` is consistent. Pattern ID format `PAT-{DOMAIN}-XXX` is consistent.
- **Agent naming**: Agent names referenced in schemas (WP27) match the names used in agent coordinator files (WP29): "2. Spec Architect", "3. Planner", "4. Coder", "5. Reviewer".
- **Test consistency**: All WPs use manual invocation testing. BDD scenarios from the spec (Section 11.2) are mapped to verification tasks (T29-09, T29-10).
- **Spec traceability**: All 16 FRs assigned. FR-001/FR-002 span 6 tasks by design (one schema file per handoff). FR-004/FR-005 span 4 tasks by design (one coordinator per agent).

---

## Spec 007 -- Docs Agent

> **Spec**: `.sdd/specs/007-docs-agent.spec.md`

### Work Packages

| ID | Title | Priority | Status | Depends On | Parallelisable |
|----|-------|----------|--------|------------|----------------|
| [WP30](WP30-foundation-doc-skills.md) | Foundation & Doc Skill Scaffolding | P0 | Not Started | none | - |
| [WP31](WP31-docs-agent-coordinator.md) | Docs Agent Coordinator | P1 | Not Started | WP30 | No |
| [WP32](WP32-technical-reference-skills.md) | Technical Reference Doc Skills | P1 | Not Started | WP30, WP31 | Yes |
| [WP33](WP33-audience-guide-skills.md) | Audience Guide Doc Skills | P1 | Not Started | WP30, WP31 | Yes |
| [WP34](WP34-code-adjacent-doc-skills.md) | Code-Adjacent Doc Skills | P1 | Not Started | WP30, WP31 | Yes |

### MVP Scope

All 5 work packages are MVP:

- WP30 (P0) creates directory scaffolding, stub skill files for all 6 doc skills, common doc skill contract, and placeholder agent file
- WP31 (P1) writes the Docs Agent coordinator with trigger handling, skill discovery, sequential dispatch, failure tolerance, pattern consumption, and commit policy
- WP32 (P1) implements doc-architecture and doc-api-reference skills (technical reference documentation)
- WP33 (P1) implements doc-user-guide and doc-developer-guide skills (audience-facing guides)
- WP34 (P1) implements doc-changelog and doc-inline-code skills (code-adjacent documentation)

The coordinator dynamically discovers skills, so WP32-WP34 integrate automatically once installed.

### Dependency & Execution Summary

- **Sequence**: WP30 -> WP31 -> {WP32, WP33, WP34}
- **Parallelization**: WP32, WP33, and WP34 can all run in parallel after WP31 completes. Each creates independent SKILL.md files.
- **Critical path**: WP30 -> WP31 -> WP34 (longest, 8 tasks)

### Sequencing Notes

WP30 creates the directory structure and stub SKILL.md files that the coordinator's dynamic discovery depends on (FR-003). Without the directories, the coordinator halts with "No doc skills found."

WP31 is the critical bottleneck: it writes the entire docs-agent.agent.md coordinator logic for context loading, skill discovery, sequential dispatch, failure tolerance, and commit policy. All skill WPs depend on the coordinator being in place.

After WP31 completes, WP32-WP34 are fully parallelizable because each creates independent SKILL.md files. WP32 (doc-architecture + doc-api-reference) is recommended first because it establishes the reference pattern for the technical-reference documentation dimension. WP33 and WP34 can follow in any order.

All implementation artifacts are markdown files (.agent.md, SKILL.md). There is no executable code, build system, or test framework. "Testing" means manually invoking the coordinator with an approved WP and verifying the output matches BDD scenarios from the spec.

### Task Index

| Task ID | Summary | Work Package | Parallel? |
|---------|---------|--------------|-----------|
| T30-01 | Create doc skill directory structure | WP30 | Yes |
| T30-02 | Create stub SKILL.md files with YAML frontmatter | WP30 | No |
| T30-03 | Create DOC-SKILL-CONTRACT.md common contract | WP30 | No |
| T30-04 | Create missing doc output files | WP30 | Yes |
| T30-05 | Create docs-agent.agent.md placeholder | WP30 | Yes |
| T30-06 | Verify directory structure and encoding compliance | WP30 | No |
| T31-01 | Write YAML frontmatter for docs-agent.agent.md | WP31 | No |
| T31-02 | Write trigger context and artifact chain loading | WP31 | No |
| T31-03 | Write dynamic skill discovery | WP31 | No |
| T31-04 | Write canonical ordering and sequential dispatch | WP31 | No |
| T31-05 | Write skill failure tolerance | WP31 | No |
| T31-06 | Write doc-patterns consumption | WP31 | Yes |
| T31-07 | Write commit policy | WP31 | No |
| T31-08 | Verify encoding compliance | WP31 | No |
| T32-01 | Create doc-architecture SKILL.md structure | WP32 | No |
| T32-02 | Write architecture docs generation sections | WP32 | No |
| T32-03 | Write incremental update logic | WP32 | No |
| T32-04 | Create doc-api-reference SKILL.md structure | WP32 | Yes |
| T32-05 | Write API reference generation from contracts | WP32 | No |
| T32-06 | Write contract-based accuracy rules | WP32 | No |
| T32-07 | Integration verification with coordinator | WP32 | No |
| T33-01 | Create doc-user-guide SKILL.md structure | WP33 | No |
| T33-02 | Write user guide generation sections | WP33 | No |
| T33-03 | Create doc-developer-guide SKILL.md structure | WP33 | Yes |
| T33-04 | Write developer guide generation sections | WP33 | No |
| T33-05 | Integration verification with coordinator | WP33 | No |
| T34-01 | Create doc-changelog SKILL.md structure | WP34 | No |
| T34-02 | Write changelog entry generation logic | WP34 | No |
| T34-03 | Write changelog prepend ordering | WP34 | No |
| T34-04 | Create doc-inline-code SKILL.md structure | WP34 | Yes |
| T34-05 | Write docstring and comment generation logic | WP34 | No |
| T34-06 | Write no-logic-modification constraint | WP34 | No |
| T34-07 | Write convention detection and adherence | WP34 | No |
| T34-08 | Integration verification with coordinator | WP34 | No |

**Total**: 5 work packages, 34 tasks

### FR Traceability

Every FR from Spec 007 is assigned to exactly one task:

| FR | Task(s) | Status |
|----|---------|--------|
| FR-001 | T31-01, T31-02 | Covered (frontmatter + trigger context) |
| FR-002 | T31-02 | Covered |
| FR-003 | T31-03 | Covered |
| FR-004 | T31-04 | Covered |
| FR-005 | T30-03, T31-04 | Covered (contract definition + dispatch) |
| FR-006 | T31-04 | Covered |
| FR-007 | T31-05 | Covered |
| FR-008 | T31-06 | Covered |
| FR-009 | T31-07 | Covered |
| FR-010 | T32-01, T32-02 | Covered |
| FR-011 | T32-03 | Covered |
| FR-012 | T32-04, T32-05 | Covered |
| FR-013 | T32-06 | Covered |
| FR-014 | T33-01, T33-02 | Covered |
| FR-015 | T33-03, T33-04 | Covered |
| FR-016 | T34-01, T34-02 | Covered |
| FR-017 | T34-03 | Covered |
| FR-018 | T34-04, T34-05 | Covered |
| FR-019 | T34-06 | Covered |
| FR-020 | T34-07 | Covered |

**FR coverage**: 20/20 FRs assigned (100%).

### Consistency Notes

Cross-WP consistency audit performed. No inconsistencies found:

- **Skill contract**: Common doc-skill contract (FR-005) defined in WP30 (DOC-SKILL-CONTRACT.md) and referenced identically in all skill WPs (WP32-WP34). Input contract has 6 fields matching FR-005.
- **Dependency graph**: No circular dependencies. WP30 -> WP31 -> {WP32 || WP33 || WP34}. All `Depends on` declarations verified valid.
- **Configuration**: Glob pattern `doc-*/SKILL.md` is consistent across coordinator (WP31) and all skill WPs. Doc output paths `.sdd/docs/` are consistent. Pattern file path `.sdd/reviews/doc-patterns.md` is consistent.
- **Canonical order**: The 6 skills' dispatch order (doc-architecture, doc-api-reference, doc-user-guide, doc-developer-guide, doc-changelog, doc-inline-code) is consistent between WP31 (coordinator) and WP32-WP34 (integration verification tasks).
- **Incremental updates**: All content doc skills (WP32, WP33) implement the incremental update pattern from FR-011. doc-changelog (WP34) uses prepend ordering (FR-017). doc-inline-code (WP34) modifies source files, not `.sdd/docs/` files -- coordinator commit policy (WP31 T31-07) handles both.
- **Test consistency**: All WPs use manual invocation testing. BDD scenarios from Section 11.2 are mapped to integration verification tasks (T32-07, T33-05, T34-08).
- **Spec traceability**: All 20 FRs (FR-001 through FR-020) are assigned. FR-001/FR-005/FR-010/FR-012/FR-014/FR-015/FR-016/FR-018 span multiple tasks by design (structure + content tasks).

---

## Spec 008 -- Orchestrator V2

> **Spec**: `.sdd/specs/008-orchestrator-v2.spec.md`

### Work Packages

| ID | Title | Priority | Status | Depends On | Parallelisable |
|----|-------|----------|--------|------------|----------------|
| [WP35](WP35-orchestrator-state-management.md) | Orchestrator V2: State File Management & Verification | P0 | Not Started | none | - |
| [WP36](WP36-orchestrator-pipeline-recovery.md) | Orchestrator V2: Pipeline Sequence, Sequential Execution & Error Recovery | P1 | Not Started | WP35 | No |
| [WP37](WP37-orchestrator-escalation-reporting.md) | Orchestrator V2: Escalation Support & Status Reporting | P1 | Not Started | WP35, WP36 | No |

### MVP Scope

All 3 work packages are MVP:

- WP35 (P0) creates the persistent state file infrastructure (.sdd/state.md) with YAML schema, initialization, update protocol, cross-verification against WP frontmatter, valid state transitions, and read-only frontmatter constraint
- WP36 (P1) rewrites the core orchestration logic with the V2 pipeline sequence (Docs Agent integration), strict sequential execution (pre-queuing bug fix), and structured error recovery (retry + escalation)
- WP37 (P1) adds universal escalation support, structured status reporting, todo list pipeline tracking, and corrupted state file recovery

All three WPs modify the same file (`orchestrator.agent.md`), so they must be implemented sequentially.

### Dependency & Execution Summary

- **Sequence**: WP35 -> WP36 -> WP37
- **Parallelization**: None -- all WPs modify the same file and have strict dependencies
- **Critical path**: WP35 -> WP36 -> WP37

### Sequencing Notes

WP35 is the foundation: it defines the state file schema and verification protocol that WP36 and WP37 depend on. Without the state file, error recovery cannot log errors, sequential execution cannot update state, and status reporting has no state to report.

WP36 is the core rewrite: it replaces the V1 pipeline sequence, decision table, workflow loop, and failure handling. It depends on WP35 because the sequential execution loop reads and writes the state file after every agent invocation, and error recovery records failures in the state file's error_log.

WP37 completes the feature set: universal escalation uses the state file to record escalation state (last_result: escalated from WP35) and the sequential execution loop to wait for resolution (from WP36). Status reporting reads state produced by the update protocol (WP35) and the pipeline/error state (WP36).

All implementation artifacts are markdown files (.agent.md). There is no executable code, build system, or test framework. "Testing" means manually invoking the Orchestrator against a workspace with WPs and verifying behavior matches BDD scenarios from Section 11.2.

### Task Index

| Task ID | Summary | Work Package | Parallel? |
|---------|---------|--------------|-----------|
| T35-01 | Define state file schema in orchestrator.agent.md | WP35 | No |
| T35-02 | Write state file creation logic | WP35 | No |
| T35-03 | Write state file update protocol | WP35 | No |
| T35-04 | Write state verification logic | WP35 | No |
| T35-05 | Write state transition validation | WP35 | Yes |
| T35-06 | Add WP frontmatter read-only constraint | WP35 | Yes |
| T35-07 | Verify state schema against companion artifacts | WP35 | No |
| T36-01 | Write updated pipeline sequence | WP36 | No |
| T36-02 | Write updated decision table | WP36 | No |
| T36-03 | Write Docs Agent delegation prompt and handoff | WP36 | Yes |
| T36-04 | Write strict sequential execution loop | WP36 | No |
| T36-05 | Write error recording and retry logic | WP36 | No |
| T36-06 | Write escalation on max retries | WP36 | No |
| T36-07 | Write review failure escalation | WP36 | Yes |
| T36-08 | Write MVP completion and pipeline halt logic | WP36 | Yes |
| T36-09 | Integration verification of pipeline and error handling | WP36 | No |
| T37-01 | Write universal escalation support | WP37 | No |
| T37-02 | Write escalation resolution logic | WP37 | No |
| T37-03 | Write status report format | WP37 | Yes |
| T37-04 | Write todo list pipeline tracker | WP37 | Yes |
| T37-05 | Write corrupted state file recovery | WP37 | Yes |
| T37-06 | Integration verification of full Orchestrator V2 | WP37 | No |

**Total**: 3 work packages, 22 tasks

### FR Traceability

Every FR from Spec 008 is assigned to exactly one task:

| FR | Task(s) | Status |
|----|---------|--------|
| FR-001 | T35-01, T35-05, T35-07 | Covered (schema + transitions + verification) |
| FR-002 | T35-02 | Covered |
| FR-003 | T35-03 | Covered |
| FR-004 | T35-04, T37-05 | Covered (verification + corrupted recovery) |
| FR-005 | T35-06 | Covered |
| FR-006 | T36-01, T36-02, T36-08 | Covered (sequence + decision table + completion) |
| FR-007 | T36-02, T36-03 | Covered (decision table + handoff) |
| FR-008 | T36-02, T36-03 | Covered (decision table + handoff) |
| FR-009 | T36-04 | Covered |
| FR-010 | T36-04 | Covered |
| FR-011 | T36-05, T36-06 | Covered (retry + escalation) |
| FR-012 | T36-07 | Covered |
| FR-013 | T36-05 | Covered |
| FR-014 | T37-01 | Covered |
| FR-015 | T37-02 | Covered |
| FR-016 | T37-03 | Covered |
| FR-017 | T37-04 | Covered |

**FR coverage**: 17/17 FRs assigned (100%).

### Consistency Notes

Cross-WP consistency audit performed. No inconsistencies found:

- **State file schema**: All 3 WPs reference the same `.sdd/state.md` schema defined in WP35 T35-01. Field names (pipeline_stage, current_spec, current_wp, last_agent, last_result, retry_count, error_log, updated_at) are used consistently.
- **Companion artifacts**: Schema matches `data-models.ts` (PipelineState, ErrorEntry interfaces) and `state-machines.ts` (VALID_PIPELINE_TRANSITIONS, MAX_RETRY_COUNT=2, MAX_REVIEW_CYCLES=3). Verified in T35-07.
- **Dependency graph**: No circular dependencies. WP35 -> WP36 -> WP37. All `Depends on` declarations verified valid.
- **File scope**: All 3 WPs modify the same file (`.github/agents/orchestrator.agent.md`) but target independent sections: WP35 adds state_schema and state_machine sections, WP36 rewrites workflow and decision table, WP37 adds escalation handling and output_format.
- **Agent naming**: Docs Agent handoff in WP36 T36-03 references the agent created by Spec 007 (docs-agent.agent.md). Pipeline agent names match existing handoff entries.
- **Constants**: retry_count max (2) is consistent between WP36 T36-05/T36-06 and companion artifact MAX_RETRY_COUNT. Review cycle max (3) is consistent between WP36 T36-07 and companion artifact MAX_REVIEW_CYCLES.
- **Test consistency**: All WPs use manual invocation testing against BDD scenarios from Section 11.2. Integration verification tasks exist in each WP (T35-07, T36-09, T37-06).

---

## Spec 009 -- Research Skill & Ideation/Brainstorming Improvements

> **Spec**: `.sdd/specs/009-research-skill-ideation.spec.md`

### Work Packages

| ID | Title | Priority | Status | Depends On | Parallelisable |
|----|-------|----------|--------|------------|----------------|
| [WP38](WP38-research-skill.md) | Research Skill | P1 | Not Started | none | No |
| [WP39](WP39-agent-integration.md) | Agent Integration | P1 | Not Started | WP38 | No |

### MVP Scope

Both work packages are MVP (US-01 is P1, US-02 is P2 but shares the same enriched brief format):

- WP38 (P1) creates the shared Research Skill at `.github/skills/research/SKILL.md` with web, codebase, and packages scope support, structured output format, and timeout handling
- WP39 (P1) integrates the Research Skill into both the Ideation Agent and Brainstorming Agent, adding enriched brief format sections (Research Findings, Risk Assessment, Technical Feasibility) and source citation requirements

### Dependency & Execution Summary

- **Sequence**: WP38 -> WP39
- **Parallelization**: None -- WP39 depends on WP38 (agents cannot dispatch a skill that does not exist yet)
- **Critical path**: WP38 -> WP39

### Sequencing Notes

WP38 creates the Research Skill file that both agents will dispatch. It must be completed first because WP39's integration verification (T39-07) validates that the dispatch prompt matches the skill's expected input format.

WP39 modifies two existing agent files. Within WP39, the ideation agent tasks (T39-01 through T39-03) and brainstorming agent dispatch task (T39-04) can partially overlap since they modify different files. However, the brainstorming enriched brief (T39-06) depends on ideation enriched brief (T39-02) being done first to establish the format pattern.

No foundation WP is needed because the only scaffolding (creating the `.github/skills/research/` directory) is trivially included in WP38's first task.

All implementation artifacts are markdown files (SKILL.md, .agent.md). There is no executable code, build system, or test framework. "Testing" means manually invoking the agents, describing an idea, and verifying the output brief structure matches BDD scenarios from the spec.

### Task Index

| Task ID | Summary | Work Package | Parallel? |
|---------|---------|--------------|-----------|
| T38-01 | Create research skill directory and SKILL.md with YAML frontmatter | WP38 | No |
| T38-02 | Write research request parameter validation section | WP38 | No |
| T38-03 | Write web scope research instructions | WP38 | Yes |
| T38-04 | Write codebase scope research instructions | WP38 | Yes |
| T38-05 | Write packages scope research instructions | WP38 | Yes |
| T38-06 | Write structured output format template | WP38 | No |
| T38-07 | Write timeout handling, error behavior, and completion constraints | WP38 | No |
| T39-01 | Add Research Skill dispatch to Ideation Agent workflow | WP39 | No |
| T39-02 | Add enriched brief format sections to Ideation Agent | WP39 | No |
| T39-03 | Add source citation requirements to Ideation Agent | WP39 | Yes |
| T39-04 | Add Research Skill dispatch to Brainstorming Agent workflow | WP39 | Yes |
| T39-05 | Add research-backed pros/cons to Brainstorming Agent | WP39 | No |
| T39-06 | Add enriched brief format to Brainstorming Agent | WP39 | No |
| T39-07 | Verify both agents dispatch Research Skill correctly | WP39 | No |

**Total**: 2 work packages, 14 tasks

### FR Traceability

Every FR from Spec 009 is assigned to exactly one task:

| FR | Task | Status |
|----|------|--------|
| FR-001 | T38-01 | Covered |
| FR-002 | T38-02 | Covered |
| FR-003 | T38-03 | Covered |
| FR-004 | T38-04 | Covered |
| FR-005 | T38-05 | Covered |
| FR-006 | T38-06 | Covered |
| FR-007 | T38-07 | Covered |
| FR-008 | T39-01 | Covered |
| FR-009 | T39-02 | Covered |
| FR-010 | T39-02 | Covered |
| FR-011 | T39-03 | Covered |
| FR-012 | T39-04 | Covered |
| FR-013 | T39-05 | Covered |
| FR-014 | T39-06 | Covered |

**FR coverage**: 14/14 FRs assigned (100%).

### Consistency Notes

Cross-WP consistency audit performed. No inconsistencies found:

- **Skill contract**: WP38 creates the Research Skill with input parameters (topic, scope, questions, output_file) matching Section 7.1 and the companion `data-models.ts` artifact. WP39 dispatches using the prompt template from Section 8.1 which maps to these same parameters.
- **Output format**: The structured output format in WP38 (T38-06) matches the ResearchOutput type from `data-models.ts`. The enriched brief sections in WP39 (T39-02, T39-06) match the EnrichedBriefSections type.
- **Dependency graph**: No circular dependencies. WP38 -> WP39. Both are P1 MVP.
- **Agent consistency**: Both agents' enriched brief formats are identical (FR-014 explicitly references FR-010). Citation format is consistent: `[Title](URL), consulted YYYY-MM-DD`.
- **Error handling**: Dispatch failure handling is consistent across both agents: log failure, proceed without research, note limitation in brief. Defaults differ appropriately: ideation uses "No research findings available" (research is automatic), brainstorming uses "No research performed" (research is on-demand).
- **Scope differences**: Ideation dispatches with scope `[web, codebase]` (FR-008); Brainstorming dispatches with scope `[web, codebase, packages]` (FR-012). This difference is intentional per spec.
- **Test consistency**: All tasks use manual invocation testing against BDD scenarios from Section 11.2.
