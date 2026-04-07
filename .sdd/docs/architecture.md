# Architecture

## System Overview

The Reviewer V2 system uses a skill-based architecture to perform multi-dimensional code reviews. It consists of two component types operating within the VS Code Copilot Chat agent framework:

1. **Review Coordinator Agent** - A lightweight dispatcher that orchestrates the review lifecycle
2. **Review Skill Files** - Self-contained review checklists executed by subagents

## Components

### Review Coordinator

**File**: `.github/agents/review-coordinator.agent.md`

The coordinator owns the entire review lifecycle but does NOT perform deep code analysis itself. Its responsibilities:

- Scope selection (accept WP ID or scan for `lane: for_review`)
- Artifact chain loading (WP plan, spec, ideation brief, plan index)
- Process compliance checks (acceptance criteria, Activity Log, commit granularity)
- Encoding checks (prohibited Unicode characters)
- Dynamic skill discovery (scan `.github/skills/review-*/SKILL.md`)
- Sequential subagent dispatch (one skill at a time via `runSubagent`)
- Findings aggregation and cross-correlation
- Verdict determination (Approved / Approved with Findings / Changes Required)
- WP lifecycle management (lane and review_status frontmatter, Activity Log)
- Review patterns curation (`.sdd/reviews/review-patterns.md`)
- Git commit of all review artifacts

The coordinator does NOT orchestrate the pipeline -- that is the Orchestrator agent's responsibility.

### Review Skills

**Path pattern**: `.github/skills/review-*/SKILL.md`

Each skill focuses on a single review domain with expert-level depth. Skills are:

| Skill | Prefix | Domain |
|-------|--------|--------|
| review-spec | SPEC- | Spec adherence (FR classification, SHALL obligations) |
| review-security | SEC- | Security (14 OWASP Secure Coding Practices categories) |
| review-quality | QUAL- | Code quality (readability, complexity, naming, style) |
| review-tests | TEST- | Test quality (validity, coverage, BDD matching) |
| review-architecture | ARCH- | Architecture adherence (components, tech stack, SOLID) |
| review-performance | PERF- | Performance (N+1 queries, blocking, caching) |
| review-docs | DOC- | Documentation accuracy (.sdd/docs/ vs implementation) |
| review-deps | DEP- | Dependency review (CVEs, licensing, supply chain) |

Skills are stateless and self-contained. Each skill file is under 300 lines and has no cross-skill dependencies.

## Interaction Flow

```
User / Orchestrator
       |
       v
Review Coordinator Agent
       |
       |--> Process compliance check (inline)
       |--> Encoding check (inline)
       |--> Discover skills (scan .github/skills/review-*/)
       |
       |--> runSubagent(review-spec)     --> writes review-spec-findings.md
       |--> runSubagent(review-security) --> writes review-security-findings.md
       |--> runSubagent(review-quality)  --> writes review-quality-findings.md
       |--> ... (for each discovered skill, sequentially)
       |
       |--> Read all findings files
       |--> Cross-correlate (merge duplicates, flag conflicts, group systemic)
       |--> Determine verdict
       |--> Write review summary to WP file
       |--> Update patterns file
       |--> Commit all artifacts
       |--> Present verdict and stop
```

## Separation of Concerns

| Concern | Owner |
|---------|-------|
| Deep code analysis | Individual review skills |
| Review orchestration | Review Coordinator |
| Pipeline orchestration | Orchestrator agent |
| Code remediation | Coder agent |
| Spec clarification | Spec Architect agent |
| Plan revision | Planner agent |

The coordinator communicates with other agents exclusively via handoff buttons -- it never invokes them directly.

## Key Design Decisions

1. **Dynamic skill discovery**: Adding a skill = creating a `.github/skills/review-*/SKILL.md` directory. No coordinator edit needed.
2. **Sequential execution**: Skills run one at a time to avoid context window contention and produce deterministic ordering.
3. **No pipeline orchestration**: The coordinator presents its verdict and stops. The Orchestrator handles next steps.
4. **Persistent findings**: Per-WP per-skill findings preserved in `.sdd/reviews/<WP-id>/` for audit trail.
5. **Active pattern curation**: FAIL findings generate patterns in `.sdd/reviews/review-patterns.md` that teach the Coder to avoid recurring mistakes.
6. **Central enum registry** (WP40): All pipeline-wide enumeration values (`lane`, `spec_status`, `pipeline_stage`, `review_status`) are defined in a single file at `.github/schemas/enums.yaml`. Agents reference this file as the authoritative source via `<!-- Enum source: .github/schemas/enums.yaml -->` comments. This eliminates scattered, potentially inconsistent inline value lists.
7. **Canonical Activity Log format** (WP40): All agents use the same log entry format: `<ISO-8601-timestamp> - <agent-name> - <action> - <details>`. The canonical template is stored in `enums.yaml` under `conventions.activity_log_format`.
8. **Structured WP frontmatter for state tracking** (WP41): Two optional YAML frontmatter fields -- `review_cycles` (integer, default 0) and `docs_completed` (boolean, default false) -- replace Activity Log text parsing for review cycle counting and documentation completion status. The Review Coordinator increments `review_cycles` on each rework cycle. The Docs Agent sets `docs_completed: true` on completion. The Orchestrator reads both fields from frontmatter instead of scanning Activity Log entries. This eliminates fragile text parsing and provides machine-readable state.
9. **Return handoff schemas** (WP42): Three return handoff schemas formalize the completion signals from Reviewer, Coder, and Docs Agent back to the Orchestrator. Each schema follows the handoff/v1 format and defines required context fields and validation rules for the return path. This completes the handoff schema coverage so every directional agent-to-agent path has an explicit contract.
10. **Shared base handoff schema** (WP42): A shared base schema at `.github/schemas/base-handoff.schema.yaml` defines reusable validation patterns (wp_file_exists, lane_value_valid, file_path_format) that individual handoff schemas reference via a `base_schema` field. If the base schema file is missing, individual schemas fall back to inline rules. This eliminates duplicated validation logic across schemas.
11. **Error-handling policy** (WP43): Pipeline agents are categorized as critical-path (HALT on failure) or advisory (best-effort). Critical-path agents (Spec Architect, Planner, Coder, Orchestrator) halt because downstream agents depend on their output for correctness. Advisory agents (Review Coordinator, Docs Agent, Ideation, Brainstorming) continue past failures because partial output is still valuable. See the "Design Decision: Error-Handling Policy" subsection below for detailed rationale per agent.
12. **Acceptance criteria maker/checker RACI** (WP43): Acceptance criteria checkboxes follow a maker/checker pattern: the Coder is Responsible (maker) for checking boxes during implementation, and the Review Coordinator is Accountable/Verifier (checker) for confirming they match actual implementation. This intentional dual-touch prevents silent regressions where a box is checked but the work is incomplete.
13. **Configurable coverage thresholds** (WP44): Two optional WP frontmatter fields -- `coverage_code` (integer 0-100, default 80) and `coverage_branch` (integer 0-100, default 90) -- allow per-WP coverage threshold overrides. The code-unit-tests, code-env-setup, and spec-test-strategy skills read these fields and fall back to defaults when absent. Each field is independent (specifying one does not require the other). A value of 0 is valid to support prototyping WPs with no coverage requirement. Out-of-range or non-integer values cause the reading skill to halt with an error.
14. **Dependency-aware WP ordering** (WP45): The Orchestrator selects the next WP for implementation using topological sort (Kahn's algorithm) over the `depends_on` dependency graph, with lowest WP number as tiebreaker. This replaces simple lowest-number-first ordering. The algorithm validates dependency references (E-051), detects circular dependencies (E-050), and reports when all WPs are blocked (E-052). WPs with no `depends_on` field or an empty array are treated as having no dependencies and are immediately eligible. When no WPs have `depends_on` fields, the sort degenerates to lowest-number-first (backward compatible). The algorithm completes in O(V+E) time.
15. **Schema versioning protocol** (WP46): Every handoff schema file in `.github/schemas/` includes a `version_history` array at the end of the file tracking all changes over time. Each entry contains `version` (string, e.g., "handoff/v1"), `date` (ISO-8601), and `description` (string). Breaking changes (removing fields, changing types, removing enum values, renaming fields) require a version increment from handoff/vN to handoff/v(N+1). Additive changes (new optional fields, new enum values, new optional rules) retain the current version but still require a version_history entry. Schemas with artifact path placeholders ({NNN}, {name}, {slug}, {NN}) include a `placeholder_patterns` section defining the regex pattern for each placeholder. The protocol is a convention enforced by review, not by runtime validation.
16. **Mid-cycle pattern propagation** (WP47): All four domain pattern files (`spec-patterns.md`, `plan-patterns.md`, `code-patterns.md`, `doc-patterns.md`) include a `patterns_version` integer in their YAML frontmatter. When the Review Coordinator adds, modifies, or retires a pattern, it increments `patterns_version` by 1 in that file. Before each skill dispatch, coordinator agents (Spec Architect, Planner, Coder, Docs Agent) compare the current `patterns_version` against their last recorded value. If the version has changed, the coordinator re-reads the full patterns file and uses the updated patterns for subsequent skill dispatches. If the patterns file is unreadable on re-check (E-031), the coordinator uses the last cached patterns and logs a warning. If frontmatter is missing, `patterns_version` is treated as 0, triggering a reload every time as a safe default (E-032). This polling mechanism enables newly discovered patterns to take effect within the same pipeline run without restarting the coordinator.

### Design Decision: Error-Handling Policy

Pipeline agents fall into two categories based on how they handle skill or step failures:

**Critical-path agents (HALT on failure)**:

| Agent | Rationale |
|-------|-----------|
| Spec Architect | Spec errors produce ambiguous or contradictory requirements that compound in every downstream artifact. Halting prevents cascading specification debt. |
| Planner | Planning errors produce incorrect task decompositions, missing contracts, or wrong dependency ordering. Downstream agents cannot recover from a flawed plan. |
| Coder | Implementation errors produce broken code that fails tests. Continuing past a failed skill (e.g., env-setup) would cause every subsequent skill to fail on the same root cause. |
| Orchestrator | The Orchestrator sequences the entire pipeline. If it cannot determine the next step or encounters an unrecoverable state, continuing risks data loss or corrupted WP state. |

**Advisory agents (best-effort on failure)**:

| Agent | Rationale |
|-------|-----------|
| Review Coordinator | Partial review output is still valuable. If one review skill fails (e.g., review-deps), the remaining skills (review-spec, review-security, etc.) still produce actionable findings. The coordinator records the failure and continues. |
| Docs Agent | Partial documentation is better than none. If one doc skill fails (e.g., doc-changelog), the other skills (doc-api-reference, doc-architecture, etc.) still produce useful output. The agent records the failure and continues. |
| Ideation | Ideation is exploratory. A failure in research or alternative generation does not invalidate the ideas already captured. The agent records the issue and presents partial results. |
| Brainstorming | Brainstorming is collaborative and iterative. A failure in one research thread does not prevent the session from producing useful output from other threads. |

This asymmetry is intentional: critical-path agents produce artifacts that downstream agents depend on for correctness, so errors must be surfaced immediately. Advisory agents produce artifacts that are consumed by humans, so partial output is preferable to no output.

## Directory Structure

```
.github/
  agents/
    brainstorming.agent.md          # Collaborative brainstorming agent
    coder.agent.md                  # Implementation coordinator
    docs-agent.agent.md             # Documentation generation agent
    ideation.agent.md               # Ideation and idea exploration agent
    orchestrator.agent.md           # Pipeline orchestration agent
    planner.agent.md                # Planning agent
    review-coordinator.agent.md     # Review orchestration agent
    spec-architect.agent.md         # Specification agent
  schemas/
    enums.yaml                      # Central enum registry (WP40) -- single source of truth for lane, spec_status, pipeline_stage, review_status, and canonical Activity Log format
    base-handoff.schema.yaml        # Shared base schema with reusable validation patterns (WP42)
    orchestrator-handoff.schema.yaml
    coder-to-reviewer.schema.yaml
    coder-complete-to-orchestrator.schema.yaml  # Return handoff: Coder -> Orchestrator (WP42)
    reviewer-to-coder.schema.yaml
    reviewer-to-orchestrator.schema.yaml        # Return handoff: Reviewer -> Orchestrator (WP42)
    reviewer-to-spec.schema.yaml
    planner-to-coder.schema.yaml
    planner-to-spec.schema.yaml
    spec-to-planner.schema.yaml
    ideation-to-spec.schema.yaml
    docs-agent-to-orchestrator.schema.yaml      # Return handoff: Docs Agent -> Orchestrator (WP42)
  skills/
    review-*/SKILL.md               # Review skills (8 dimensions)
    code-*/SKILL.md                 # Coding skills (5 phases)
    doc-*/SKILL.md                  # Documentation skills (6 types)
    semantic-commit/SKILL.md        # Commit grouping skill

.sdd/
  ideas/                            # Ideation briefs
  specs/                            # Specifications
  plans/                            # Work packages and contracts
  reviews/
    review-patterns.md              # Active + resolved review patterns
    doc-patterns.md                 # Active + resolved doc patterns
    <WP-id>/                        # Per-WP findings directory
  docs/                             # Generated documentation
```

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Agent framework | VS Code Copilot Chat agents (`.agent.md` files) |
| Skill framework | VS Code Copilot Chat skills (`SKILL.md` files) |
| Data format | Markdown with YAML frontmatter |
| Version control | Git (via terminal commands, explicit `git add`) |
| File system | Local workspace only -- no external services |

---

## Coder V2 - Skill-Based Coordinator

The Coder V2 replaces the monolithic single-pass implementation agent with a lightweight coordinator that dispatches 5 sequential coding skills. The coordinator writes no implementation code itself.

### Coder Coordinator

**File**: `.github/agents/coder.agent.md`

The coordinator owns the entire WP implementation lifecycle:

- WP selection (from argument or user prompt)
- Artifact chain loading (WP, spec, contracts, AGENTS.md, README)
- Dependency verification (prior WPs must have `lane: done`)
- Contract file validation (all referenced contracts must exist)
- Code patterns consumption (`.sdd/reviews/code-patterns.md`)
- Dynamic skill discovery (scan `.github/skills/code-*/SKILL.md`)
- Sequential skill dispatch via `runSubagent`
- Conditional debug dispatch with 3-attempt retry
- Task state tracking (`manage_todo_list`, acceptance criteria checkboxes)
- Coverage verification (configurable per-WP via `coverage_code`/`coverage_branch` frontmatter, defaults 80%/90%)
- Per-task commits with explicit file listing
- Handoff to Reviewer (no self-review)

### Coding Skills

**Path pattern**: `.github/skills/code-*/SKILL.md`

Skills execute in canonical order:

| Order | Skill | Phase |
|-------|-------|-------|
| 1 | `code-env-setup` | Environment verification, dependency installation |
| 2 | `code-implementation` | Contract-first task implementation |
| 3 | `code-unit-tests` | Unit test writing and execution |
| 4 | `code-integration-tests` | Integration test writing and execution |
| 5 | `code-debug` | Conditional: test failure diagnosis and fix (max 3 attempts) |

All coding skills follow the common contract defined in `.github/skills/CODER-SKILL-CONTRACT.md` (FR-017 through FR-019).

### Coder Interaction Flow

```
User / Orchestrator
       |
       v
Coder Coordinator
       |
       |--> Read WP + contracts + spec + patterns
       |--> Verify dependencies (prior WPs done)
       |--> Verify contract files exist
       |--> Discover skills (scan .github/skills/code-*/)
       |
       |--> runSubagent(code-env-setup)          --> env verified
       |--> runSubagent(code-implementation)      --> code written
       |--> runSubagent(code-unit-tests)          --> unit tests written + run
       |--> runSubagent(code-integration-tests)   --> integration tests written + run
       |
       |--> Check test results
       |     |--> All pass? --> coverage check --> for_review --> handoff to Reviewer
       |     |--> Fail? --> runSubagent(code-debug) (max 3x) --> re-check
       |                          |--> Still fail after 3? --> escalate to human
       |
       |--> Commit per task
       |--> Set lane: for_review
       |--> Handoff to Review Coordinator
```

### Key Coder Design Decisions

1. **No self-review**: The Reviewer is the sole quality gate. The Coder's job ends at "tests pass + coverage met".
2. **Contract files are read-only**: Produced by the Planner, consumed verbatim by the Coder. Implementation must match contracts exactly.
3. **One implementation skill per WP**: All tasks in a WP are handled by a single `code-implementation` invocation.
4. **Debug skill with 3-attempt budget**: Balances autonomy with human escalation. After 3 failed debug attempts, escalates to the user.
5. **Dynamic skill discovery**: Adding a coding phase = creating a `.github/skills/code-*/SKILL.md` directory. No coordinator edit needed.
