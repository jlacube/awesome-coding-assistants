# SDD Pipeline V2: Universal Skill Architecture - Brainstorming Brief

## The Idea

Transform the entire Spec-Driven Development (SDD) pipeline from a collection of monolithic agents into a unified architecture where every agent follows the same pattern: a lightweight coordinator dispatching sequential, context-forwarding skills. This is the same pattern that made Reviewer V2 successful, applied universally to the Spec Architect, Planner, Coder, and a new Docs Agent. The goal is a fully autonomous pipeline that produces functional software aligned with specifications, with human gates only at spec and plan approval.

## Problem & Opportunity

### The Current Pain

The SDD pipeline works when a human is in the loop to catch drift. In autonomous mode, three handoff points consistently fail:

1. **Spec Architect -> Planner**: The spec is prose-heavy and shallow. The Planner cannot reliably decompose it because:
   - Tasks end up too coarse or too fine
   - Acceptance criteria are vague
   - Implementation context is missing
   - Task boundaries create coupling
   - WPs are constrained to ~300 lines but suffer from ALL quality dimensions

2. **Planner -> Coder**: Tasks say WHAT but not HOW. Missing data model and interface information forces the Coder to improvise, leading to spec drift. The Coder needs formal contracts (interfaces, schemas, signatures) to implement faithfully.

3. **Coder -> Reviewer**: The Reviewer catches real spec drift and finds gaps the spec never addressed. The Coder's self-review misses issues the Reviewer finds, creating a false sense of security. Fix cycles loop because the root cause is upstream (spec and plan quality), not code quality.

### Root Cause Chain

```
Spec (prose-heavy, shallow) -> no formal artifacts
    |
Planner (poor decomposition) -> tasks lack concrete references
    |
Coder ("WHAT not HOW") -> drifts, improvises
    |
Reviewer (catches drift) -> but also finds spec gaps
    |
Fix cycle -> patches symptoms, not root causes
```

### The Opportunity

The Reviewer V2 refactoring proved that decomposing a monolithic agent into a coordinator + focused skills:
- Eliminates context overload (each skill gets fresh context)
- Improves depth (each skill focuses on one dimension)
- Enables extensibility (add a skill = add a capability)
- Maintains consistency (coordinator enforces output format)

This same pattern can solve the problems across the entire pipeline. Additionally, introducing formal contract artifacts (interfaces, schemas, signatures in the target language) at the planning stage gives the Coder concrete blueprints rather than prose interpretations.

## Competitive Landscape

| System | Approach | Strengths | Weaknesses | Differentiation |
|--------|----------|-----------|------------|----------------|
| **Devin (Cognition)** | Single autonomous agent | End-to-end from issue to PR; sandboxed compute | No formal spec phase; single context window; no structured review | SDD has formal specs, contracts, and multi-dimensional review |
| **OpenHands** | SDK + agent framework | Composable agents; SDK for custom workflows; enterprise support | Task-oriented (fix issue), not spec-driven; no pipeline decomposition | SDD provides structured pipeline with human gates and contracts |
| **SWE-agent** | LLM + tool use for GitHub issues | State-of-art on SWE-bench; configurable YAML; open source | Issue-to-code only; no planning, spec, or review phases | SDD covers full lifecycle from idea to documented software |
| **GitHub Copilot Workspace** | Plan-implement-review in IDE | Integrated into GitHub; plan step before coding | No formal contracts; limited review depth; no autonomous loop | SDD has 36 specialized review/implementation skills |
| **Cursor/Windsurf** | AI-augmented editor | Fast iteration; inline suggestions | Human in the loop always; no autonomous pipeline | SDD automates the full loop with escalation only when needed |

**Key differentiation**: No existing system combines (1) formal specification with contracts, (2) skill-based agent decomposition, (3) autonomous pipeline with human gates, and (4) comprehensive multi-dimensional review. The SDD pipeline occupies a unique position as a "specification-faithful autonomous development system."

## Failed Predecessors

| Attempt | What They Tried | Why It Failed | Lesson for SDD |
|---------|----------------|---------------|---------------|
| **Monolithic Reviewer (SDD V1)** | Single 350+ line agent covering 13 review dimensions | Context overload; missed issues; wrong priorities; 3 bullets for security vs 14 OWASP categories | Decompose into focused skills with fresh context per dimension |
| **Auto-GPT (2023-2024)** | Fully autonomous agent loops with no human gates | Infinite loops; goal drift; no convergence; wasted tokens | Human gates at spec + plan prevent unbounded divergence |
| **Early code generation tools** | LLM generates code from natural language description | No verification that code matches intent; accumulated drift | Formal contracts provide verifiable alignment between spec and code |

## Vision

A development pipeline where:
- A human describes an idea and approves two checkpoints (spec and plan)
- Everything else runs autonomously: specification deepening, contract generation, task decomposition, implementation, testing, review, documentation, and iteration
- Every agent follows the same architectural pattern (coordinator + skills), making the system maintainable, extensible, and debuggable
- Formal contracts in the target language serve as the single source of truth that connects spec to code to review to documentation
- When the pipeline cannot resolve an issue autonomously, it escalates to the human with full context rather than guessing

## Target Users

**Primary**: Developers using VS Code with GitHub Copilot who want to build software from specifications rather than ad-hoc coding. They trust the pipeline to produce correct, documented, tested software from their ideas.

**Secondary**: Teams adopting spec-driven development who need the pipeline agents and skills as a reusable framework for their own projects.

## Core Value Proposition

Turn a validated specification into fully functional, tested, documented software with zero human intervention after plan approval - and know that every line of code traces back to a formal contract derived from the spec.

## Key Capabilities

### P1 - Must-Have (MVP)

Each of these is independently deliverable and testable:

- **Spec Architect V2 (skill-based)**: Coordinator + 8 sequential skills producing deep spec with companion artifacts (data models, API contracts, interfaces, state machines, error catalogs) in the target language. 80% of technical design lives in the spec. Skills use shared file context forwarding.

- **Planner V2 (skill-based)**: Coordinator + 8 sequential skills producing WP task lists AND language-specific contract files. Two-phase approach: plan first, then contracts per WP in 800-line blocks. Auto-loops to Spec Architect when gaps found. Contracts stored in `.sdd/plans/contracts/`.

- **Coder V2 (skill-based, simplified)**: Coordinator + 5 sequential skills (env setup, core implementation, unit tests, integration tests, debugging on failure). Self-review removed. Contract-first implementation: copies interface definitions verbatim, implements against API contracts, validates against data schemas.

- **review-spec-completeness skill**: Pre-planning validation gate. Checks: all FR have SHALL statements, data models have field-level types, API contracts present, error codes cataloged, state machines defined, security requirements per component, traceability matrix complete.

- **Agent-to-agent handoff schemas**: Formalized contracts in `.github/schemas/` defining what each agent produces and what the next agent expects. Eliminates interpretation-based drift at handoff points.

### P2 - Important (next increment)

- **Docs Agent (NEW, skill-based)**: Coordinator + 6 skills (architecture docs, API reference from contracts, user guide, developer guide, changelog/release notes, inline code docs). Runs after each WP is reviewed and approved. Dedicated agent for documentation rather than burdening the Coder.

- **Orchestrator V2**: State file (`.sdd/state.md`) + WP frontmatter verification for cross-session progress tracking. Error recovery with retry for failed agents. Fix pre-queuing bug (strict sequential execution: one agent, re-read state, then decide next). Any agent can escalate to human.

- **Domain-specific patterns**: Replace single `review-patterns.md` with per-domain files (`spec-patterns.md`, `plan-patterns.md`, `code-patterns.md`, `doc-patterns.md`). Each agent reads patterns relevant to its domain.

- **Reviewer V2 enhancement**: Existing review-spec skill expands to validate code against formal contracts (interfaces, schemas, API endpoints), not just prose spec text.

### P3 - Nice-to-Have (future)

- **Research Skill (shared)**: Web search + codebase analysis + package registry lookups. Shared by Ideation and Brainstorming agents for competitive analysis, technology evaluation, and pain point discovery.

- **Ideation/Brainstorming improvements**: Both agents gain access to Research Skill. Brief output format deepened to provide better input for Spec Architect V2.

- **Pipeline analytics**: Track success rates per agent, per skill. Identify which skills most frequently produce FAILs. Optimize the pipeline based on data.

## Decision Log

Major decisions made during the brainstorming session with rationale and alternatives considered.

| # | Decision | Chosen Approach | Alternatives Considered | Rationale |
|---|----------|----------------|------------------------|-----------|
| 1 | Who produces formal contracts? | Planner generates contracts (80% Spec, 20% Planner) | A: New Contract Designer agent; C: Spec Architect + validation gate | Planner already understands decomposition; no new handoff point; Spec provides 80% of technical design |
| 2 | Contract format | Language-specific code (TypeScript, Python, etc.) | Structured markdown; pseudo-code; language-agnostic types | Eliminates translation/interpretation step; Coder can copy verbatim |
| 3 | Planning approach for context limits | Two-phase: plan first, then contracts per WP in 800-line blocks | One mega-plan; spec adds all depth | Manages context window limits; maintains quality per block |
| 4 | Spec depth vs Planner contracts | 80% Spec Architect, 20% Planner fills gaps | 50/50 split; Planner owns all technical design; new agent | Spec is source of truth; Planner refines with implementation-specific details |
| 5 | Coder self-review | Remove entirely | Keep but improve; replace with contract validation; keep + gate | Reviewer is purpose-built; self-review creates false security; trust the dedicated system |
| 6 | Universal agent architecture | Every agent = Coordinator + Sequential Skills | Only Reviewer uses skills; some agents use skills; none | Reviewer V2 pattern proven successful; consistency; maintainability; context management |
| 7 | Context forwarding between skills | Shared accumulator file (each skill r/w same file) | Prompt injection; separate files per skill then merge; coordinator passes prior output | Most natural for sequential workflows; each skill sees current state |
| 8 | Documentation ownership | New dedicated Docs Agent (skill-based) | Coder updates during implementation; Reviewer triggers updates; auto-generate only | Separates concerns; Coder focuses on code; docs quality needs dedicated attention |
| 9 | Orchestrator state tracking | State file + WP frontmatter verification | State file only; read frontmatter only | Redundancy catches inconsistencies; state file enables cross-session continuity |
| 10 | Escalation model | Any agent can escalate to human | Only Orchestrator escalates; per-domain escalation | Democratic; fastest path to resolution; agents closest to the problem escalate directly |
| 11 | Spec validation gate | New review skill (review-spec-completeness) | Built into Planner; standalone agent; Spec Architect self-validates | Fits existing review skill architecture; reusable; focused |
| 12 | Planner autonomy on spec gaps | Auto-loops to Spec Architect | Flags to human; tries to resolve alone | Reduces human intervention; Spec Architect is the domain expert for gaps |
| 13 | Contract-aware review | review-spec skill expands to check contracts | New review-contracts skill; each review skill checks relevant contracts | No skill proliferation; contract checking is spec adherence by nature |
| 14 | Implementation scope | Multiple specs per agent | One unified spec for everything | Each agent refactoring is independently deliverable and reviewable |
| 15 | Patterns structure | One file per domain | Single file with tags; embedded in agent instructions | Clean separation; each agent reads only its domain; avoids context bloat |
| 16 | Contract artifact location | `.sdd/plans/contracts/` | `.sdd/contracts/`; inline in WP files; `.sdd/schemas/` | Under plans because contracts are part of planning output; keeps plan self-contained |
| 17 | Agent handoff schema location | `.github/schemas/` | `.sdd/schemas/`; embedded in agent files; single file | Always distributed with agents; part of agent infrastructure, not project-specific |
| 18 | Coder skill architecture | Sequential: setup -> impl -> unit tests -> integration -> debug | Per-task; hybrid | Clean pipeline; each skill gets fresh context; debugging only on failure |
| 19 | Research Skill scope | Web + codebase + package registries | Web only; web + codebase | Full scope enables technology decisions, dependency choices, and competitive analysis |

## Out of Scope

- **Runtime/deployment infrastructure**: The pipeline produces code and docs; deployment automation is separate.
- **IDE integration beyond VS Code Copilot Chat**: The agents and skills are designed for the VS Code Copilot Chat agent framework. Other editors are not considered.
- **Multi-repository workflows**: The pipeline operates within a single workspace/repo.
- **Real-time collaboration**: The pipeline is single-user, single-pipeline at a time.
- **Custom LLM provider support**: Uses whatever model the VS Code Copilot Chat provides.
- **CI/CD pipeline generation**: The Coder implements code; CI/CD setup is a separate concern.

## Assumptions & Risks

### Assumptions

1. The VS Code Copilot Chat agent framework supports the skill dispatch patterns needed (sequential invocation, context forwarding via files).
2. Context windows are large enough for skills to read shared accumulator files + their own instructions.
3. Language-specific contract generation is reliable enough across target languages (TypeScript, Python, etc.).
4. The Planner can auto-loop to Spec Architect without creating infinite recursion (needs max iteration guard).
5. ~36 skills is maintainable as a single repository of `.github/skills/` directories.

### Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Context forwarding breaks with large specs | Medium | High | 800-line block limit; skills read only relevant sections of accumulator |
| 36 skills become a maintenance burden | Medium | Medium | Consistent skill template; each skill independently testable; clear naming |
| Skills produce inconsistent output formats | Medium | High | Agent handoff schemas enforce format; coordinator validates output |
| Orchestrator pre-queuing bug resurfaces | High | Medium | Strict rule: one agent, re-read state, then next; architectural guardrail |
| Contract artifacts get stale vs code during fix cycles | Low | High | Reviewer validates contracts against code; contracts are source of truth |
| Planner-Spec Architect auto-loop cycles forever | Low | High | Max 3 iterations; escalate to human if unresolved after 3 loops |

## Technical Feasibility

### Confirmed Feasible

- **Skill-based agent decomposition**: Proven with Reviewer V2 (8 skills, sequential dispatch, dynamic discovery, cross-correlation)
- **Shared file context forwarding**: Skills already write findings files that the coordinator reads; same pattern for accumulator files
- **Agent handoff via `runSubagent`**: VS Code Copilot Chat supports subagent invocation with fresh context
- **Dynamic skill discovery**: Glob pattern scanning (`review-*/SKILL.md`) already works in Reviewer V2

### Needs Validation

- **Sequential skills with context forwarding**: Reviewer V2 skills are independent; dependent sequential skills (where skill N reads skill N-1's output) need testing
- **Language-specific contract generation quality**: LLMs generate code well, but generating precise interface definitions from prose specs needs validation at scale
- **Cross-session state persistence**: The Orchestrator state file needs to survive across VS Code sessions reliably
- **800-line block iteration**: The Planner generating contracts in capped blocks needs testing to ensure consistency across blocks

### Platform Constraints

- **VS Code Copilot Chat agent framework**: `.agent.md` files with YAML frontmatter; `SKILL.md` files discovered via glob patterns
- **Tool availability**: `runSubagent`, `read_file`, `create_file`, `replace_string_in_file`, `run_in_terminal`, `grep_search`, `fetch_webpage`
- **Data format**: Markdown with YAML frontmatter for all SDD artifacts; no database or external storage

## Open Questions

1. **Skill ordering within Spec Architect**: What is the optimal sequence for the 8 spec skills? Data model first? Requirements first? Does order matter if they use shared file forwarding?
2. **Contract versioning**: When the spec is revised mid-pipeline, how do contracts stay in sync? Does the Planner regenerate all contracts or only affected ones?
3. **Docs Agent triggering**: Does the Docs Agent run after every WP approval, or once all WPs for a spec are done?
4. **Skill granularity in Coder**: Is "core implementation" one skill invocation per WP (potentially large), or should it be decomposed further per-task?
5. **Research Skill integration**: How does the Research Skill hand findings to the Ideation/Brainstorming agents? Inline in prompt or via file artifacts?
6. **Spec-specific agent schemas**: Should handoff schemas vary by project type (web app vs CLI vs library), or be universal?
7. **Patterns consumption**: When should agents read their domain patterns - at startup, before each skill, or on-demand?
8. **Testing the pipeline**: How do we test the pipeline itself? Integration tests that run a sample idea through the full pipeline?

## Session Summary

- **Rounds of Q&A**: 11 (10 structured question rounds + synthesis rounds)
- **Topics explored**: Current agent gaps, spec generation improvements, automation bottlenecks, documentation lifecycle, artifact generation, inter-agent handoff reliability, orchestrator intelligence, coder self-sufficiency, planner decomposition quality, feedback loops, universal skill architecture
- **Alternatives generated**: 19 major decision points with 2-3 alternatives each (57+ alternatives total)
- **Key pivot points**:
  1. Shifted from "improve specs" to "Planner generates formal contracts" based on user preference for contract ownership
  2. Shifted from "Spec Architect writes everything" to "80/20 split" between Spec and Planner
  3. Bold decision to REMOVE Coder self-review entirely (trust the Reviewer)
  4. Universal adoption of Reviewer V2 skill pattern for ALL agents (not just Reviewer)
  5. Changed from "one unified spec" to "multiple specs per agent" for independent delivery
- **Research conducted**: OpenHands, SWE-agent, Devin capabilities, contract-first API design (Swagger/OpenAPI), architecture decision records (ADR), arc42 architecture documentation, fitness functions for decisions

## Proposed Spec Decomposition

Since the decision is to create multiple specs, here is the proposed breakdown:

| Spec # | Scope | Dependencies | Priority |
|--------|-------|-------------|----------|
| 002 | Spec Architect V2 (skill-based, deep spec generation) | None | P1 - First |
| 003 | Planner V2 (skill-based, contract generation) | Spec Architect V2 | P1 - Second |
| 004 | Coder V2 (skill-based, contract-first, no self-review) | Planner V2 | P1 - Third |
| 005 | review-spec-completeness skill + review-spec expansion | Spec Architect V2, Planner V2 | P1 - Fourth |
| 006 | Agent handoff schemas + patterns restructuring | All above | P1 - Fifth |
| 007 | Docs Agent (NEW, skill-based) | Coder V2 | P2 |
| 008 | Orchestrator V2 (state tracking, error recovery) | All above | P2 |
| 009 | Research Skill + Ideation/Brainstorming improvements | None (independent) | P3 |

## Next Step

Hand off to the Spec Architect agent to translate this brief into formal specifications, starting with Spec 002 (Spec Architect V2). Each spec should follow the established 16-section template with the deeper technical design approach defined in this brief.
