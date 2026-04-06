# SDD Pipeline Hardening -- Ideation Brief

## The Idea

A targeted hardening pass on the SDD pipeline that addresses 14 viability concerns discovered during a thorough codebase assessment. Rather than redesigning the architecture, this initiative introduces structured frontmatter fields, a central enum registry, missing handoff schemas, a documented error-handling policy, and configurable thresholds -- all minimal-change improvements that eliminate fragile couplings and undocumented conventions while preserving the existing agent/skill architecture.

## Problem & Opportunity

The SDD pipeline (8 agents, 37 skills, 8 handoff schemas) successfully built itself through 9 specs and 39 WPs. However, a systematic viability assessment revealed 14 concerns spanning five themes:

1. **Fragile coupling to Activity Log format** (Concerns 1-3): Three agents parse Activity Log text with different format expectations. The Orchestrator counts review cycles by scanning log entries, and determines documentation completion the same way. Any format change breaks both.

2. **Undocumented conventions and ambiguous lifecycles** (Concerns 4-5, 9): Spec status transitions (Draft/Validated/Approved, plus a phantom "Final" value), error-handling asymmetry (upstream HALT vs downstream best-effort), and lane values are all implicit knowledge -- understood by the developers who built the system but not formalized anywhere a new contributor or the system itself can reliably reference.

3. **Missing structural guarantees** (Concerns 6, 8, 10-11): WP numbering does not guarantee dependency order, schemas lack versioning rules, no shared base schema exists, and return handoff paths (reviewer-to-orchestrator, coder-to-orchestrator) have no explicit schema definitions.

4. **Dual ownership and hardcoded values** (Concerns 12-13): Both Coder and Review Coordinator claim responsibility for acceptance criteria checkboxes. Coverage thresholds (80%/90%) are hardcoded in skills rather than sourced from spec or WP configuration.

5. **Operational gaps** (Concerns 7, 14): Contract files remain untested against real source code (only `.gitkeep` placeholders exist), and pattern file changes do not propagate within a running pipeline cycle.

**Cost of not solving**: As the pipeline builds larger, real-world projects (not just itself), these fragilities compound. Review cycle miscounts trigger false escalations or missed escalations. Undocumented error policies confuse users when security reviews silently fail. Hardcoded thresholds force spec workarounds. Missing schemas allow malformed handoffs to pass silently.

## Competitive Landscape

This is an internal infrastructure improvement to a bespoke pipeline. No external competitors apply directly. The closest analogues are:

- **GitHub Actions / Azure Pipelines**: Use structured YAML with explicit schema validation for every step. This initiative brings the SDD pipeline closer to that standard by formalizing handoff schemas and enum registries.
- **Terraform state management**: Terraform solves a similar "state tracking" problem by using a single structured state file rather than parsing log output. The frontmatter-field approach (Concerns 2-3) follows this pattern.

[Unverified] No established open-source "AI agent pipeline" projects were found that address the specific problem of multi-agent handoff schema consistency in LLM-driven development pipelines.

## Vision

After this hardening pass, the SDD pipeline:
- Tracks review cycles and documentation completion via explicit frontmatter fields, not log parsing
- Has a single source of truth for all enum values (lane, spec status, pipeline stage)
- Validates every agent-to-agent handoff against an explicit schema -- including return paths
- Documents its error-handling policy as a deliberate design decision
- Allows specs to override default coverage thresholds
- Resolves dual ownership of acceptance criteria checkboxes with clear RACI

## Target Users

- **Pipeline operators**: Developers using the SDD pipeline to build software. They need predictable behavior, clear error messages, and correct escalation logic.
- **Pipeline maintainers**: Anyone modifying agents, skills, or schemas. They need documented conventions, central enums, and explicit contracts so changes do not silently break cross-agent behavior.

## Core Value Proposition

Eliminate fragile implicit couplings and undocumented conventions so the pipeline behaves predictably as it scales beyond self-referential builds to real-world codebases.

## Key Capabilities

### P1 -- Must-Have (MVP)

- **C1: Structured WP frontmatter for review and docs tracking** (Concerns 2, 3): Add `review_cycles: N` and `docs_completed: true/false` fields to WP frontmatter. The Review Coordinator increments `review_cycles` on each review round. The Docs Agent sets `docs_completed: true` on completion. The Orchestrator reads these fields instead of scanning Activity Logs. Activity Logs remain as human-readable audit trails but are no longer the source of truth for state queries.

- **C2: Central enum registry** (Concerns 4, 9): Create a single file (`.github/schemas/enums.yaml`) defining all enum values used across the pipeline:
  - `lane`: planned, doing, for_review, to_do, done, blocked
  - `spec_status`: Draft, Validated, Approved (remove phantom "Final" reference from Planner)
  - `pipeline_stage`: idle, ideation, specification, planning, implementation, review, documentation, complete
  - `review_status`: pending, has_feedback, acknowledged, approved
  All agents and schemas reference this file as the single source of truth. Validation rules in schemas use `$ref` to this enum file.

- **C3: Standardized Activity Log format** (Concern 1): Define a single canonical format for all agents:
  `<ISO-8601-timestamp> - <agent-name> - <action> - <details>`
  Document this in the enum registry or a companion conventions file. Update Coder and Review Coordinator to use the same format. The Orchestrator no longer parses Activity Logs for state -- it reads frontmatter (C1) -- so format consistency is for human readability and auditability, not machine parsing.

- **C4: Missing return handoff schemas** (Concern 11): Create:
  - `.github/schemas/reviewer-to-orchestrator.schema.yaml`: Defines what the Review Coordinator passes back (verdict, WP path, updated lane value)
  - `.github/schemas/coder-complete-to-orchestrator.schema.yaml`: Defines what the Coder signals on WP completion (WP path, lane=for_review confirmation)
  - `.github/schemas/docs-agent-to-orchestrator.schema.yaml`: Defines docs completion signal
  These are lightweight schemas following the existing `handoff/v1` pattern.

- **C5: Documented error-handling policy** (Concern 5): Add a "Design Decisions" section to the architecture documentation (`.sdd/docs/architecture.md`) explicitly documenting the error-handling asymmetry:
  - Critical-path agents (Spec Architect, Coder) HALT on failure because errors compound downstream
  - Advisory agents (Review Coordinator, Docs Agent) use best-effort because partial output is still valuable
  - Add a brief note to each agent file referencing this policy so users can find the rationale

- **C6: Acceptance criteria RACI** (Concern 12): Clarify dual responsibility:
  - Coder: checks off acceptance criteria checkboxes during implementation (responsible)
  - Review Coordinator: verifies checkboxes match actual implementation (accountable/verifier)
  - This is not redundant -- it is a maker/checker pattern. Document this in both agent files and the developer guide.

### P2 -- Important (next increment)

- **C7: Configurable coverage thresholds** (Concern 13): Add optional `coverage_code: N` and `coverage_branch: N` fields to WP frontmatter (or spec-level configuration). Skills read these values, falling back to the current defaults (80%/90%) when not specified. This allows specs to override thresholds without modifying skill files.

- **C8: WP dependency-aware ordering** (Concern 6): Modify the Orchestrator's WP selection logic from "lowest-numbered first" to "topological sort by dependency graph, then lowest-numbered as tiebreaker." The Planner already records dependencies in WP frontmatter (`depends_on`). The Orchestrator should use these to build a correct execution order rather than relying on numbering.

- **C9: Schema versioning protocol** (Concern 8): Document schema versioning rules:
  - `handoff/v1` to `handoff/v2`: breaking change, old agents must be updated
  - Minor additions (new optional fields): stay on same version
  - Add a `version_history` section to each schema file
  - Add regex validation for artifact path placeholders (`{NNN}` = `\d{2,3}`, `{name}` = `[a-z0-9-]+`, `{slug}` = `[a-z0-9-]+`)

- **C10: Shared schema base** (Concern 10): Extract common validation patterns (WP file existence, lane field check, file path validation) into a shared base schema (`.github/schemas/base-handoff.schema.yaml`). Individual schemas inherit from or reference this base. Reduces duplication and ensures consistent validation rules.

### P3 -- Nice-to-Have (future)

- **C11: Pattern file propagation mechanism** (Concern 14): Add a `patterns_version` counter to the patterns file. Before each skill dispatch, the coordinator checks if `patterns_version` has changed since the last read and reloads if so. This is an optimization -- the current behavior (patterns take effect next invocation) is correct, just suboptimal for long pipeline runs.

- **C12: Contract files validation pilot** (Concern 7): Create a simple integration test that runs the Coder against a small, real codebase (not markdown) to validate that contract-first implementation, coverage enforcement, and debug retry loops work with actual source code. This is a confidence-building exercise, not a feature.

## Out of Scope

- **Full pipeline redesign**: This is a hardening pass, not a rewrite. Agent architecture, skill dispatch patterns, and the overall pipeline flow remain unchanged.
- **Runtime validation tooling**: No new CLI tools or automated schema validators are created. Validation remains agent-driven (agents read schemas and check conditions).
- **Multi-language schema formats**: Schemas stay in YAML. No JSON Schema, OpenAPI, or other format conversions.
- **Parallel WP execution**: The pipeline remains sequential. Concern 6 is addressed through better ordering, not parallelism.
- **Pattern file real-time push**: No event system or file-watching mechanism. C11 uses a poll-on-dispatch approach.

## Assumptions & Risks

| # | Assumption | Risk if Wrong |
|---|-----------|---------------|
| 1 | Adding frontmatter fields to WP files is backward-compatible -- existing WP files without the new fields are handled gracefully by agents checking for field presence | Agents could fail on older WP files lacking new fields |
| 2 | A single `enums.yaml` file is sufficient -- no agent needs runtime-generated enum values | If an agent dynamically creates lane values, the registry becomes stale |
| 3 | The existing `handoff/v1` schema format is sufficient for return handoff schemas | Return handoffs may need different structure than forward handoffs |
| 4 | Standardizing Activity Log format does not break any existing parsing logic | If any tooling outside the pipeline parses logs, it could break |
| 5 | Coverage threshold overrides in frontmatter will be correctly propagated to test runners by skills | Skills may have hardcoded paths that ignore frontmatter values |

## Research Findings

- [VS Code Copilot Chat Extensibility - Custom Instructions](https://code.visualstudio.com/docs/copilot/copilot-customization), consulted 2026-04-06. Confirms that agent and skill files use markdown with YAML frontmatter, validating that adding frontmatter fields is the idiomatic extension mechanism for this platform.
- [YAML Ain't Markup Language (YAML) Version 1.2 Specification](https://yaml.org/spec/1.2.2/), consulted 2026-04-06. Confirms YAML supports anchors, aliases, and merge keys that could enable the shared base schema approach (C10) without custom tooling.

[Unverified] The "maker/checker" pattern (C6) is a standard internal control principle from financial auditing. No specific source cited for its application to AI agent acceptance criteria.

## Risk Assessment

| Risk | Likelihood | Impact | Source |
|------|-----------|--------|--------|
| Frontmatter field additions break older WP files | low | medium | Codebase assessment -- all WP files are generated by the pipeline and can be migrated |
| Enum registry becomes a bottleneck for changes | low | low | Single file with clear ownership; changes are infrequent |
| Standardized log format breaks unknown downstream consumers | low | low | No external consumers identified in codebase assessment |
| Schema versioning rules are too rigid for rapid iteration | medium | medium | No prior versioning experience in this codebase |
| Coverage threshold override ignored by hardcoded skill logic | medium | high | Skills currently have thresholds as literal values in multiple locations |

## Technical Feasibility

| Item | Status | Evidence | Source |
|------|--------|----------|--------|
| WP frontmatter field additions | Confirmed feasible | All agents already read/write YAML frontmatter in WP files | Codebase: coder.agent.md, review-coordinator.agent.md |
| Central enum YAML file | Confirmed feasible | Existing schemas use YAML format; agents already read schema files | Codebase: .github/schemas/*.schema.yaml |
| Return handoff schemas | Confirmed feasible | 8 existing schemas follow a consistent pattern that can be replicated | Codebase: .github/schemas/ |
| Configurable coverage thresholds | Needs validation | Skills have hardcoded values in multiple locations; refactoring to read from frontmatter needs testing | Codebase: code-env-setup/SKILL.md, code-unit-tests/SKILL.md |
| Dependency-aware WP ordering | Confirmed feasible | WP frontmatter already contains `depends_on` field; Orchestrator already checks dependencies | Codebase: orchestrator.agent.md, data-models.ts |
| Shared base schema | Needs validation | YAML merge keys could work but need testing with the agents' schema-reading logic | YAML 1.2 spec |

## Open Questions

1. Should the enum registry be a standalone YAML file or embedded in a broader "pipeline conventions" document?
2. Should return handoff schemas be full schemas or lightweight "signal" schemas with minimal fields?
3. For coverage threshold overrides, should the override live in the WP frontmatter, the spec file, or a project-level config file?
4. Should the Activity Log format standardization include a machine-parseable structured section (e.g., YAML block) alongside the human-readable line, or is a single standardized text format sufficient?
5. Should contract file validation (C12) be a dedicated WP or an acceptance criterion within the overall hardening spec?

## Next Step

Hand off to the Spec Architect agent to translate this brief into a formal specification covering all 14 concerns as a single coherent improvement initiative.
