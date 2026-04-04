# Reviewer V2 - Skill-Based Review Architecture - Brainstorming Brief

## The Idea

Redesign the monolithic Reviewer agent (currently 350+ lines covering 13 review dimensions) into a lightweight Review Coordinator agent that dispatches 8 specialized review skills via subagents. Each skill focuses on a single review domain with deep expertise, solving the current agent's context overload, missed issues, and maintainability problems. The coordinator aggregates findings, cross-correlates across dimensions, manages the review lifecycle, and curates a patterns file that teaches the Coder agent to avoid recurring mistakes.

## Problem & Opportunity

The current "5. Reviewer" agent tries to do everything in a single pass:
- 13 review dimensions in one agent file
- Pipeline orchestration (auto-continuation, handoffs to Coder/Planner/Spec Architect)
- WP lifecycle management (lane transitions, activity logs)
- Report generation with 13-dimension statistics table

**Observed problems:**
- **Misses important issues** - context window pressure from loading all artifacts + all code + 13 checklists means the agent checks superficially across all dimensions rather than deeply in any one
- **Wrong priorities** - treats encoding checks and documentation accuracy with the same weight as spec adherence and security vulnerabilities
- **Hard to maintain** - 350+ lines of instructions in a single file; changing one dimension risks breaking others
- **Security gap** - current security section has 3 bullet points; OWASP Secure Coding Practices has 14 categories with 100+ checklist items
- **Missing critical dimensions** - readability/complexity (Google's #3 review dimension), error handling, naming quality, dependency review, code style/consistency, and concurrency checks are all absent

**What currently exists and why it is insufficient:**
The current agent is functional but monolithic. It was designed when the SDD pipeline was simpler. As the codebase and specifications grow, the reviewer's effectiveness degrades because it cannot maintain depth across all dimensions simultaneously.

**Real user frustration (from session):** "maybe lacking security review, worried about agent size" and "wrong priorities (checks trivial things, misses core)"

## Competitive Landscape

| Solution | Strengths | Weaknesses | Differentiation Opportunity |
|----------|-----------|------------|---------------------------|
| Google Eng Practices | Well-researched, covers design/functionality/complexity/tests/naming/comments/style/docs/context | Human-oriented - not structured for AI agents; no security depth | Translate their dimension taxonomy into AI-executable skills |
| Microsoft 30 Best Practices (Dr. Michaela Greiler) | Research-backed, covers both author and reviewer practices; emphasizes checklists and starting with test code | Process-focused rather than checklist-focused; not structured for automated review | Adopt "start with test code" and checklist-driven approach |
| Codacy Automated Review | 10-category checklist (logic, error handling, readability, structure, security, performance, coverage, deps, standards, docs) | Tool-specific; shallow security (no OWASP depth); focused on linting-level checks | Go deeper than automated tools by combining AI understanding with structured checklists |
| OWASP Code Review Guide | Deep security focus, 14 categories, 100+ items; industry standard | Security-only - no quality, performance, or spec adherence | Use as the foundation for the Security skill |
| Stack Overflow / Gergely Orosz "Good vs Better Reviews" | Distinguishes surface-level from deep contextual review; emphasizes system-level thinking | Conceptual guide, not actionable checklist | Implement "better review" patterns: contextual analysis, system-level thinking |
| Swarmia Guide | Practical PR workflow; emphasizes speed, CI integration, explicit communication | GitHub PR-specific; no deep technical review dimensions | Adopt their "focus on the right things" prioritization philosophy |

## Failed Predecessors

- **Monolithic review agents** - The current approach itself is the "failed predecessor." As review dimensions multiply, a single agent's effectiveness decreases. This is the motivating problem.
- **Overly strict automated review tools** - Tools that flag hundreds of style violations without prioritization lead to "alert fatigue" - developers stop reading the output. Research from Microsoft confirms "the quality and value of code review feedback decrease with the size of the change" - the same applies to review instructions.

## Vision

A review system where:
- Every code change gets deep, expert-level scrutiny across 8 specialized dimensions
- Security vulnerabilities are caught against the full OWASP checklist, not a 3-bullet summary
- The Coder agent learns from past review patterns and avoids repeating mistakes
- Review findings are precisely actionable, cross-correlated across dimensions, and never superficial
- Adding or modifying a review dimension means editing one focused skill file, not a 350-line monolith

## Target Users

**Primary: The Coder Agent (autonomous pipeline)**
Receives review findings as structured FB-XX items that are immediately actionable. Benefits from the patterns file that prevents recurring mistakes.

**Secondary: Human Developers**
Read review reports in WP files and detailed findings in `.sdd/reviews/<WP-id>/` for audit purposes. Need the findings to be precise, cited, and non-ambiguous.

**Tertiary: The Orchestrator Agent**
Needs to know the review verdict (Approved/Changes Required) to route the pipeline. No longer relies on the reviewer for pipeline orchestration.

## Core Value Proposition

A modular, deeply thorough code review system that catches what monolithic reviewers miss by giving each review dimension its own focused context and expert-level checklist, while cross-correlating findings across dimensions to surface systemic issues.

## Key Capabilities

### P1 - Must-Have (MVP)

- **Review Coordinator Agent**: Lightweight dispatcher that loads WP + Spec, dispatches subagent reviews, aggregates findings with cross-correlation, writes summary+verdict+FB-XX to WP file, manages WP lifecycle (lane, review_status, activity log), curates patterns file, and commits
- **Spec Adherence Skill (review-spec)**: Reviews every FR-XXX, data model, API contract, and success criteria against the specification. Flags Missing/Partial/Deviating/Compliant per requirement. Checks preconditions, postconditions, error paths, edge cases.
- **Security Skill (review-security)**: Full 14-category OWASP Secure Coding Practices audit: input validation, output encoding, authentication, session management, access control, cryptographic practices, error handling/logging, data protection, communication security, system configuration, database security, file management, memory management, general coding practices. Skills should skip categories that don't apply to the codebase.
- **Code Quality Skill (review-quality)**: Readability assessment, complexity analysis, naming quality, comment quality (explain "why" not "what"), error handling patterns, style/consistency with codebase, dead code detection, duplication detection.

### P2 - Important (next increment)

- **Test Quality Skill (review-tests)**: Test validity (can tests actually fail?), coverage thresholds (80% code, 90% branch), BDD scenario matching, edge case coverage, error path testing, test naming, Arrange/Act/Assert structure, no vacuous assertions or fully-mocked subjects.
- **Architecture Skill (review-architecture)**: Component/design adherence to spec, technology stack compliance, directory structure compliance, separation of concerns, SOLID principles (SRP especially), dependency direction, scope discipline (no scope creep, no extra files).

### P3 - Nice-to-Have (future)

- **Performance Skill (review-performance)**: N+1 query detection, index analysis, async/blocking patterns, unbounded data fetching, unnecessary computation in hot paths, efficient data structures, caching opportunities.
- **Documentation Skill (review-docs)**: Architecture docs vs real structure, API docs vs actual endpoints, configuration docs vs real env vars, data model docs vs actual schema, user/developer/deployment guide accuracy, no stale content.
- **Dependencies Skill (review-deps)**: Known CVEs in dependencies, abandoned/unmaintained packages, unnecessary dependencies, license compatibility, pinned versions, supply chain integrity (lockfiles/checksums).

## Decision Log

| Decision | Chosen Approach | Alternatives Considered | Rationale |
|----------|----------------|------------------------|-----------|
| Architecture model | 1 coordinator + 8 skill subagents | Monolith (current), skills-in-coordinator, 3-4 grouped agents, many small agents | Fresh context window per dimension avoids overload; maintainable focused files |
| Pipeline coupling | Pure reviewer - remove pipeline orchestration | Keep orchestration, lightweight handoff only | Separation of concerns; Orchestrator agent owns pipeline flow |
| Review philosophy | Adversarial + WARN tier for non-critical | Pure auditor (no WARN), constructive reviewer, tiered by priority | FAILs for critical deviations; WARNs for non-blocking issues that should be reviewed |
| Skill loading | Always load all 8 skills for every review | Smart selection based on context, user-chosen per review | Thoroughness over speed; no dimension should be skipped |
| Execution model | Sequential subagents (each with fresh context) | Parallel subagents, sequential skills in coordinator, two-pass scan | VS Code subagents are sequential; fresh context per skill solves overload |
| Execution speed | Thoroughness over speed (20-40 min acceptable) | Quick scan + deep dive, time budgets per skill | User explicitly chose depth over speed at every decision point |
| Security depth | Full 14-category OWASP audit | Lightweight (Top 10 only), medium (systematic subset) | Security was flagged as a major gap in the current reviewer |
| OWASP relevance | Skills skip categories that don't apply to the codebase | Coordinator filters categories, run all and accept N/A | Avoids wasting time on memory management checks for a web app |
| Feedback mechanism | Patterns file (structured checklist) at .sdd/reviews/review-patterns.md | Feed back into Coder instructions, no feedback, auto-generated notes | Coder reads patterns before implementing; curator keeps it manageable |
| Patterns lifecycle | Coordinator curates - removes resolved, keeps active | Hard cap at 20 items, rolling last 5 WPs, append-only | Prevents staleness without arbitrary limits |
| Report structure | WP gets summary+verdict+FB-XX; detailed findings in .sdd/reviews/WP-id/ | Single consolidated report in WP, summary in WP + details elsewhere | Keeps WP file manageable; per-skill findings preserved for audit |
| Temp findings format | .sdd/reviews/WP-id/skill-findings.md (persistent) | Temp directory (deleted), in-memory via subagent return | Per-WP organization; preserved for reference and audit trail |
| Coordinator intelligence | Cross-correlation across dimensions | Simple aggregation, full narrative synthesis | Catches conflicts (e.g., Security PASS but Code Quality finds eval()) |
| Duplicate findings | Coordinator merges related findings into composite issues | Keep all duplicates, deduplicate only | Same issue flagged by 2 skills becomes one composite finding with dual perspective |
| Verdict logic | Any FAIL in any skill = Changes Required | Only P1 FAILs block, weighted scoring | Strict bar - consistent with current agent's approach |
| Re-review scope | FAILed skills + skills whose files were modified by fixes | All 8 every time, only FAILed skills | Efficient but catches regressions from fixes |
| Process compliance | Coordinator responsibility (not a skill) | 9th skill, merged into Spec Adherence | Process checks are meta-level, not code review |
| Scope discipline | Part of Architecture skill | Coordinator, Spec Adherence, standalone skill | Scope = architectural concern (what belongs where) |
| Encoding checks | Coordinator responsibility | Code Quality skill, drop entirely | Quick check, not worth a full skill invocation |
| Agent name | "5. Review Coordinator" | "5. Reviewer" (current), "5. Code Reviewer" | Reflects the dispatching/aggregating role |
| Handoffs | Fix Findings (Coder) + Spec/Plan gaps (Architect/Planner) | Fix Findings only, keep all 4 current handoffs | Removed "Implement Next WP" since pipeline orchestration is gone |
| Orchestrator updates | Out of scope - flagged as dependency | In scope, ignore | Focused effort; Orchestrator update is a separate task |
| Coder integration | Document expectation; Coder update is separate | Modify Coder in this effort, auto-generated notes | Keeps scope manageable; patterns file is the integration point |
| Skill naming | review-spec, review-security, review-quality, review-architecture, review-tests, review-performance, review-docs, review-deps | Various alternatives | Consistent "review-" prefix; descriptive suffix |
| Code context | Each subagent discovers and reads code independently | Coordinator pre-reads and passes, coordinator sends file list | Simpler architecture; each subagent has full autonomy |
| Finding caps | No cap - thoroughness is the goal | Cap at 10-15 most critical, group by severity | Consistent with "thoroughness over speed" philosophy |

## Out of Scope

- **Modifying the Coder agent** to read the patterns file (documented as expectation, implemented separately)
- **Updating the Orchestrator agent** to remove dependency on reviewer's pipeline orchestration (flagged as dependency)
- **Modifying the Planner or Spec Architect agents** (only the reviewer is being redesigned)
- **Language-specific review rules** (skills are language-agnostic, processes and patterns only)
- **CI/CD integration** (review happens within VS Code agent framework, not as a CI pipeline step)

## Assumptions & Risks

### Assumptions
- VS Code's subagent mechanism provides sufficient context window per subagent for deep review
- Sequential subagent execution (20-40 minutes total) is acceptable for the target workflow
- The Coder agent will be updated separately to read the patterns file
- The Orchestrator agent will be updated separately to handle pipeline flow without reviewer's auto-continuation
- Skills are language-agnostic and can be applied to any technology stack
- Each subagent can independently discover and read relevant code files without coordinator pre-analysis

### Risks (from pre-mortem)
- **Context window still insufficient**: Even with focused skills, complex codebases may exceed a single subagent's context capacity. Mitigation: skills are designed to be concise (50-150 lines each); code discovery is targeted.
- **8 sequential passes are too slow**: 20-40 minutes per review may be a bottleneck in rapid iteration. Mitigation: re-reviews are scoped (only FAILed + modified); user explicitly accepted this tradeoff.
- **Patterns file staleness**: Without active curation, the patterns file could accumulate 100+ items and become noise. Mitigation: coordinator curates after each review, removing resolved patterns.
- **Cross-correlation is shallow**: Without the code context that subagents had, the coordinator may struggle to meaningfully correlate findings. Mitigation: subagents write structured findings to persistent files; coordinator reads all before synthesizing.
- **Skill maintenance drift**: 8 skill files could become inconsistent over time. Mitigation: skills contain only checklists and rules, no workflow logic; coordinator owns the workflow.
- **Orchestrator breaking change**: Removing pipeline orchestration from the reviewer requires an Orchestrator update. If not done simultaneously, the pipeline could stall after a review. Mitigation: flagged as explicit dependency; Orchestrator update should be prioritized alongside this effort.

## Technical Feasibility

### Agent/Skill Architecture
- The coordinator agent is a `.agent.md` file (`.github/agents/review-coordinator.agent.md`)
- Each review skill is a `SKILL.md` file under `.github/skills/review-<dimension>/SKILL.md`
- Subagents are invoked via `runSubagent` with a prompt that tells them to read the relevant SKILL.md file
- Findings are written to `.sdd/reviews/<WP-id>/<skill>-findings.md`
- Patterns file lives at `.sdd/reviews/review-patterns.md`

### Data Flow
```
1. Coordinator receives review request (WP ID or scans for lane: for_review)
2. Coordinator loads WP + Spec chain (brief, spec, WP plan)
3. Coordinator checks process compliance (activity logs, checklists)
4. Coordinator checks encoding (UTF-8 violations)
5. Coordinator creates .sdd/reviews/<WP-id>/ directory
6. Coordinator dispatches 8 subagent reviews sequentially:
   - Each subagent: reads SKILL.md -> discovers code -> reviews -> writes findings file
7. Coordinator reads all 8 findings files
8. Coordinator cross-correlates: merges duplicate findings, flags conflicts
9. Coordinator writes summary+verdict+FB-XX to WP file
10. Coordinator updates WP frontmatter (lane, review_status)
11. Coordinator updates .sdd/reviews/review-patterns.md (curate: add new, remove resolved)
12. Coordinator commits
```

### Constraints
- VS Code subagents run sequentially (blocking), not in parallel
- Each subagent gets a fresh context window - no shared state between skills
- Tools available to subagents: read_file, search (text, semantic, file), list_dir, web/fetch
- Coordinator needs: runSubagent, file operations, askQuestions, todo, web

### File Structure
```
.github/
  agents/
    review-coordinator.agent.md    (replaces reviewer.agent.md)
  skills/
    review-spec/SKILL.md           (P1)
    review-security/SKILL.md       (P1)
    review-quality/SKILL.md        (P1)
    review-architecture/SKILL.md   (P2)
    review-tests/SKILL.md          (P2)
    review-performance/SKILL.md    (P3)
    review-docs/SKILL.md           (P3)
    review-deps/SKILL.md           (P3)

.sdd/
  reviews/
    review-patterns.md             (curated patterns checklist)
    WP01-feature-name/
      review-spec-findings.md
      review-security-findings.md
      review-quality-findings.md
      review-architecture-findings.md
      review-tests-findings.md
      review-performance-findings.md
      review-docs-findings.md
      review-deps-findings.md
```

## Open Questions

- How should the Coder agent's instructions reference the patterns file? (Needs to be defined when modifying the Coder)
- Should the Orchestrator be updated before or simultaneously with the reviewer redesign?
- What is the minimum viable skill file size? Should skills be kept under 100 lines?
- Should skills include language-specific extensions (e.g., Python-specific security checks)?
- How should the re-review prompt differ from the initial review prompt for subagents?
- Should the coordinator support "partial reviews" (e.g., review only P1 dimensions for a quick check)?
- What happens when a new skill is added later (e.g., accessibility review)? Is the coordinator designed for extensibility?

## Session Summary

- Rounds of Q&A: 10
- Topics explored: Architecture model, review philosophy, skill granularity, pipeline coupling, security depth, feedback mechanism, patterns file design, execution model, report structure, pre-mortem scenarios, priority ranking, coordinator responsibilities, handoff design, skill taxonomy, cross-correlation strategy, verdict logic, re-review scoping
- Alternatives generated: 40+ across all decision points
- Key pivot points:
  - Round 3: User chose subagent execution over skills-in-coordinator, fundamentally changing the architecture from "one big agent" to "dispatcher + specialists"
  - Round 4: "Always load all 8" combined with "subagents" cemented the deep-review-over-speed philosophy
  - Round 6: Temporary findings files evolved to persistent per-WP findings files for audit trail
  - Round 7: Pre-mortem surfaced the cross-correlation challenge, leading to the structured findings file approach

## Next Step

Hand off to the Spec Architect agent to translate this brief into a formal specification covering:
1. The Review Coordinator agent specification
2. 8 individual skill specifications with detailed checklists
3. The patterns file format specification
4. The findings file format specification
5. Integration points with Coder and Orchestrator agents
