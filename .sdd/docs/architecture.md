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

## Directory Structure

```
.github/
  agents/
    orchestrator.agent.md           # Pipeline orchestration agent
    coder.agent.md                  # Implementation coordinator
    review-coordinator.agent.md     # Review orchestration agent
    docs-agent.agent.md             # Documentation generation agent
    planner.agent.md                # Planning agent
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
- Coverage verification (80% code, 90% branch)
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
