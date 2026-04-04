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

## Directory Structure

```
.github/
  agents/
    review-coordinator.agent.md     # Coordinator agent
    reviewer.agent.md.deprecated    # Old monolithic reviewer (kept for reference)
  skills/
    review-spec/SKILL.md            # Spec adherence skill
    review-security/SKILL.md        # Security skill
    review-quality/SKILL.md         # Code quality skill
    review-tests/SKILL.md           # Test quality skill
    review-architecture/SKILL.md    # Architecture skill
    review-performance/SKILL.md     # Performance skill
    review-docs/SKILL.md            # Documentation skill
    review-deps/SKILL.md            # Dependency skill

.sdd/
  reviews/
    review-patterns.md              # Active + resolved patterns
    <WP-id>/                        # Per-WP findings directory
      review-spec-findings.md       # One findings file per dispatched skill
      review-security-findings.md
      ...
```

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Agent framework | VS Code Copilot Chat agents (`.agent.md` files) |
| Skill framework | VS Code Copilot Chat skills (`SKILL.md` files) |
| Data format | Markdown with YAML frontmatter |
| Version control | Git (via terminal commands, explicit `git add`) |
| File system | Local workspace only -- no external services |
