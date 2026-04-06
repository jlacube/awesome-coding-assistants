# SDD Pipeline Hardening -- Specification

> **Source brief**: `.sdd/ideas/003-sdd-pipeline-hardening.md`
> **Feature branch**: `010-sdd-pipeline-hardening`
> **Status**: Validated
> **Version**: 1.0

---

## 1. Overview

A targeted hardening pass on the SDD pipeline that addresses 14 viability concerns discovered during a codebase assessment of the 8-agent, 37-skill, 8-schema pipeline. This initiative introduces structured WP frontmatter fields for review and documentation tracking, a central enum registry, missing return handoff schemas, a documented error-handling policy, configurable coverage thresholds, dependency-aware WP ordering, schema versioning protocol, and shared schema infrastructure -- all minimal-change improvements that eliminate fragile couplings and undocumented conventions while preserving the existing agent/skill architecture. Deliverables are markdown instruction files (.agent.md, SKILL.md), YAML schema files (.schema.yaml), and skill contract files -- not executable source code.

---

## 2. Goals & Success Criteria

- **SC-001**: The Orchestrator reads review cycle counts and documentation completion status from WP frontmatter fields, not from Activity Log text parsing. Verified by: Orchestrator agent instructions reference frontmatter fields `review_cycles` and `docs_completed` instead of Activity Log scanning logic.
- **SC-002**: All enum values used across the pipeline (lane, spec_status, pipeline_stage, review_status) are defined in a single central registry file. Verified by: `.github/schemas/enums.yaml` exists and all agent/schema files reference it as the authoritative source.
- **SC-003**: Every agent-to-agent handoff -- including return paths (reviewer-to-orchestrator, coder-complete-to-orchestrator, docs-agent-to-orchestrator) -- has a corresponding schema file in `.github/schemas/`. Verified by: file count matches total directional handoff paths.
- **SC-004**: The error-handling policy (critical-path HALT vs advisory best-effort) is explicitly documented in `.sdd/docs/architecture.md` and referenced from each agent file. Verified by: architecture doc contains a "Design Decisions" subsection on error handling; each agent file contains a reference to it.
- **SC-005**: All agents writing Activity Log entries use the same canonical format: `<ISO-8601> - <agent-name> - <action> - <details>`. Verified by: all agent files specify the same format string.
- **SC-006**: Acceptance criteria checkbox ownership follows a documented maker/checker pattern (Coder checks, Reviewer verifies). Verified by: both agent files and the developer guide document this RACI.
- **SC-007**: Coverage thresholds (code and branch) can be overridden per-WP via optional frontmatter fields, with skills falling back to defaults (80%/90%) when not specified. Verified by: skill instructions reference frontmatter fields with fallback logic.
- **SC-008**: The Orchestrator selects WPs using topological sort by dependency graph (from `depends_on` frontmatter) with lowest-numbered as tiebreaker, not purely by WP number. Verified by: Orchestrator instructions describe the topological sort algorithm.
- **SC-009**: Each schema file includes version history metadata and follows documented versioning rules (major bump for breaking changes, same version for additive optional fields). Verified by: versioning protocol documented and schema files contain `version_history` sections.
- **SC-010**: Common validation patterns are extracted into a shared base schema referenced by individual handoff schemas. Verified by: `.github/schemas/base-handoff.schema.yaml` exists and is referenced by at least two other schemas.
- **SC-011**: A pattern file propagation mechanism exists so mid-cycle pattern updates take effect on the next skill dispatch within the same pipeline run. Verified by: coordinator agents check `patterns_version` before each skill dispatch.
- **SC-012**: A contract file validation pilot confirms that contract-first implementation works against real source code. Verified by: validation test exists and documents results.

---

## 3. Users & Roles

- **Pipeline Operator**: A developer using the SDD pipeline to build software projects. Needs predictable behavior, correct escalation logic, and clear error messages. Uses the pipeline end-to-end but does not modify agent or skill files. Affected by: C1 (reliable review tracking), C5 (clear error policy), C7 (threshold overrides), C8 (correct WP ordering).
- **Pipeline Maintainer**: Anyone modifying agents, skills, schemas, or skill contracts. Needs documented conventions, central enum definitions, explicit contracts, and version history so changes do not silently break cross-agent behavior. Affected by: C2 (enum registry), C3 (log format), C4 (return schemas), C6 (RACI), C9 (schema versioning), C10 (shared base schema).
- **Orchestrator Agent**: The automated coordinator that drives the pipeline cycle. Consumes WP frontmatter, handoff schemas, and enum values to make routing decisions. Affected by: C1 (frontmatter fields), C2 (enum values), C4 (return schemas), C8 (dependency ordering).
- **Review Coordinator Agent**: The automated reviewer that dispatches review skills and manages WP lifecycle. Consumes handoff schemas, writes review cycle data. Affected by: C1 (writes review_cycles), C3 (log format), C6 (checker role).
- **Coder Agent**: The automated implementer that dispatches coding skills. Consumes coverage thresholds and manages acceptance criteria. Affected by: C3 (log format), C6 (maker role), C7 (reads threshold overrides).
- **Docs Agent**: The automated documentation generator. Signals docs completion. Affected by: C1 (writes docs_completed), C3 (log format).

---

## 4. Functional Requirements

### 4.1 Structured WP Frontmatter for Review and Docs Tracking (C1)

- **FR-001**: WP file YAML frontmatter SHALL support an optional `review_cycles` field of type integer with a default value of 0.
  - Precondition: WP file exists with valid YAML frontmatter.
  - Postcondition: WP frontmatter schema accepts `review_cycles` as a valid optional field.
  - Error: If `review_cycles` is present but not a non-negative integer, the reading agent SHALL treat it as 0 and log a warning.

- **FR-002**: The Review Coordinator agent SHALL increment `review_cycles` by 1 in WP frontmatter each time it sets a WP's lane to `to_do` (rework requested).
  - Precondition: Review Coordinator has completed review and determined verdict is "Changes Required."
  - Postcondition: `review_cycles` value in WP frontmatter is incremented by 1 alongside the lane change to `to_do`.
  - Error: If `review_cycles` field is absent, the Review Coordinator SHALL add it with value 1.

- **FR-003**: WP file YAML frontmatter SHALL support an optional `docs_completed` field of type boolean with a default value of false.
  - Precondition: WP file exists with valid YAML frontmatter.
  - Postcondition: WP frontmatter schema accepts `docs_completed` as a valid optional field.
  - Error: If `docs_completed` is present but not a boolean, the reading agent SHALL treat it as false and log a warning.

- **FR-004**: The Docs Agent SHALL set `docs_completed: true` in WP frontmatter upon successful completion of documentation generation for that WP.
  - Precondition: Docs Agent has been invoked for a WP with `lane: done` and all doc skills have completed (or failed with best-effort).
  - Postcondition: WP frontmatter contains `docs_completed: true`.
  - Error: If the WP file cannot be written, the Docs Agent SHALL log the error and report it in its completion signal.

- **FR-005**: The Orchestrator SHALL read `review_cycles` from WP frontmatter to count review cycles for escalation decisions, replacing the current Activity Log scanning approach.
  - Precondition: Orchestrator is evaluating whether to escalate a WP that has been returned from review.
  - Postcondition: Escalation decision is based on `review_cycles >= 3` from frontmatter, not Activity Log entry counting.
  - Error: If `review_cycles` is absent or unparseable, the Orchestrator SHALL treat it as 0 (no escalation).

- **FR-006**: The Orchestrator SHALL read `docs_completed` from WP frontmatter to determine documentation status, replacing the current Activity Log scanning approach.
  - Precondition: Orchestrator is determining whether a WP with `lane: done` has been documented.
  - Postcondition: Documentation status decision is based on `docs_completed == true` from frontmatter.
  - Error: If `docs_completed` is absent or unparseable, the Orchestrator SHALL treat it as false (not yet documented).

- **FR-007**: All agents reading `review_cycles` or `docs_completed` SHALL treat absent fields as their default values (0 and false, respectively) to maintain backward compatibility with existing WP files.
  - Precondition: Agent reads a WP file that was created before these fields were introduced.
  - Postcondition: Agent proceeds with default values without errors.
  - Error: None -- this FR defines graceful degradation behavior.

#### Implementation Contract -- Structured WP Frontmatter

**Inputs**: WP file path (string); YAML frontmatter fields `review_cycles` (optional integer), `docs_completed` (optional boolean)
**Outputs**: Updated WP file with modified frontmatter fields
**Error behaviors**:
- Missing `review_cycles` field -> treat as 0, add field on first write
- Missing `docs_completed` field -> treat as false, add field on first write
- Non-integer `review_cycles` value -> treat as 0, log warning
- Non-boolean `docs_completed` value -> treat as false, log warning
- WP file write failure -> log error, do not halt (for advisory agents), halt (for critical-path agents)

**Files modified**:
- `.github/agents/orchestrator.agent.md` -- replace Activity Log scanning with frontmatter reads
- `.github/agents/review-coordinator.agent.md` -- add `review_cycles` increment logic
- `.github/agents/docs-agent.agent.md` -- add `docs_completed` write logic
- `.github/agents/coder.agent.md` -- no changes (Coder does not track review cycles or docs)

---

### 4.2 Central Enum Registry (C2)

- **FR-008**: A central enum registry file SHALL exist at `.github/schemas/enums.yaml`.
  - Precondition: `.github/schemas/` directory exists.
  - Postcondition: `enums.yaml` file exists and is valid YAML.
  - Error: If the file is missing, agents referencing it SHALL halt with: "Enum registry not found at .github/schemas/enums.yaml."

- **FR-009**: The enum registry SHALL define named enum groups for: `lane`, `spec_status`, `pipeline_stage`, and `review_status`.
  - Precondition: `enums.yaml` exists.
  - Postcondition: File contains four top-level keys, each mapping to an array of valid values.
  - Error: If a required enum group is missing, the validating agent SHALL halt with: "Enum group '<name>' not found in enums.yaml."

- **FR-010**: The `lane` enum group SHALL define exactly these values: `planned`, `doing`, `for_review`, `to_do`, `done`, `blocked`.
  - Precondition: `enums.yaml` contains a `lane` key.
  - Postcondition: `lane` array contains exactly the six listed values, no more, no fewer.
  - Error: If a lane value not in this set is encountered, the consuming agent SHALL halt with: "Invalid lane value '<value>'. Valid values: planned, doing, for_review, to_do, done, blocked."

- **FR-011**: The `spec_status` enum group SHALL define exactly these values: `Draft`, `Validated`, `Approved`.
  - Precondition: `enums.yaml` contains a `spec_status` key.
  - Postcondition: `spec_status` array contains exactly the three listed values.
  - Error: If a spec status not in this set is referenced, the consuming agent SHALL halt with: "Invalid spec_status '<value>'. Valid values: Draft, Validated, Approved."

- **FR-012**: The `pipeline_stage` enum group SHALL define exactly these values: `idle`, `ideation`, `specification`, `planning`, `implementation`, `review`, `documentation`, `complete`.
  - Precondition: `enums.yaml` contains a `pipeline_stage` key.
  - Postcondition: `pipeline_stage` array contains exactly the eight listed values.
  - Error: If a pipeline_stage not in this set is used, the Orchestrator SHALL halt with: "Invalid pipeline_stage '<value>'."

- **FR-013**: The `review_status` enum group SHALL define exactly these values: `pending`, `has_feedback`, `acknowledged`, `approved`.
  - Precondition: `enums.yaml` contains a `review_status` key.
  - Postcondition: `review_status` array contains exactly the four listed values.
  - Error: If a review_status not in this set is used, the consuming agent SHALL halt with: "Invalid review_status '<value>'."

- **FR-014**: All agent files that reference enum values SHALL include a comment citing `enums.yaml` as the authoritative source for those values.
  - Precondition: Agent files currently hardcode enum values inline.
  - Postcondition: Each agent file contains a comment near its enum references: `<!-- Enum source: .github/schemas/enums.yaml -->`.
  - Error: If enums.yaml is not referenced, there is no runtime error, but the spec compliance check will flag it.

- **FR-015**: The Planner agent instructions SHALL NOT reference a "Final" spec status value. Any existing reference SHALL be removed.
  - Precondition: Planner agent file may contain references to "Final" as a spec status.
  - Postcondition: No references to "Final" as a spec status exist in the Planner agent file.
  - Error: If "Final" references remain after implementation, the review SHALL flag it as a spec compliance violation.

#### Implementation Contract -- Central Enum Registry

**Inputs**: None (new file creation)
**Outputs**: `.github/schemas/enums.yaml` containing four enum groups with defined values
**Error behaviors**:
- enums.yaml missing at agent startup -> agent halts with descriptive error
- Unknown enum value encountered -> agent halts with error listing valid values
- Malformed YAML in enums.yaml -> agent halts with YAML parse error

**Files modified**:
- `.github/schemas/enums.yaml` -- new file
- `.github/agents/orchestrator.agent.md` -- add enums.yaml reference comments
- `.github/agents/coder.agent.md` -- add enums.yaml reference comments
- `.github/agents/review-coordinator.agent.md` -- add enums.yaml reference comments
- `.github/agents/docs-agent.agent.md` -- add enums.yaml reference comments
- `.github/agents/planner.agent.md` -- remove "Final" reference, add enums.yaml reference
- `.github/agents/spec-architect.agent.md` -- add enums.yaml reference comments

---

### 4.3 Standardized Activity Log Format (C3)

- **FR-016**: All agents writing Activity Log entries in WP files SHALL use the canonical format: `<ISO-8601-timestamp> - <agent-name> - <action> - <details>`.
  - Precondition: Agent is about to append an entry to a WP file's Activity Log section.
  - Postcondition: The appended entry matches the canonical format exactly.
  - Error: If the Activity Log section does not exist in the WP file, the agent SHALL create it before appending.

- **FR-017**: The Coder agent instructions SHALL specify the canonical Activity Log format defined in FR-016, replacing any existing format specification.
  - Precondition: Coder agent file exists with an Activity Log protocol section.
  - Postcondition: The Activity Log protocol in the Coder agent file uses the canonical format.
  - Error: If the existing format section cannot be located, the implementer SHALL add the canonical format at the appropriate location.

- **FR-018**: The Review Coordinator agent instructions SHALL specify the canonical Activity Log format defined in FR-016.
  - Precondition: Review Coordinator agent file exists.
  - Postcondition: Any Activity Log format instructions in the Review Coordinator use the canonical format.
  - Error: Same as FR-017.

- **FR-019**: The Docs Agent instructions SHALL specify the canonical Activity Log format defined in FR-016 for any log entries it writes.
  - Precondition: Docs Agent file exists.
  - Postcondition: Docs Agent log entry format uses the canonical format.
  - Error: Same as FR-017.

- **FR-020**: The canonical Activity Log format SHALL be documented in `.github/schemas/enums.yaml` or a companion conventions section so that all agents reference a single format definition.
  - Precondition: enums.yaml exists (per FR-008).
  - Postcondition: enums.yaml contains a `conventions` section with the canonical log format string.
  - Error: If conventions section is missing, agents fall back to inline format definition.

#### Implementation Contract -- Activity Log Format

**Inputs**: Activity log entry components: ISO-8601 timestamp (string), agent name (string), action (string), details (string)
**Outputs**: Formatted log entry string: `<timestamp> - <agent-name> - <action> - <details>`
**Error behaviors**:
- Missing Activity Log section in WP file -> create section heading before appending
- Malformed timestamp -> use current time in ISO-8601 format

**Files modified**:
- `.github/agents/coder.agent.md` -- update Activity Log Protocol section
- `.github/agents/review-coordinator.agent.md` -- update log format references
- `.github/agents/docs-agent.agent.md` -- add/update log format
- `.github/schemas/enums.yaml` -- add conventions section with canonical format

---

### 4.4 Return Handoff Schemas (C4)

- **FR-021**: A schema file SHALL exist at `.github/schemas/reviewer-to-orchestrator.schema.yaml` defining the Review Coordinator's completion signal to the Orchestrator.
  - Precondition: `.github/schemas/` directory exists.
  - Postcondition: Schema file exists and follows the handoff/v1 format.
  - Error: If the file is missing, the Orchestrator's schema validation step SHALL report it.

- **FR-022**: The reviewer-to-orchestrator schema SHALL require these context fields: `wp_path` (string, required), `verdict` (enum: approved or changes_required, required), and `updated_lane` (enum from lane values, required).
  - Precondition: Schema file exists per FR-021.
  - Postcondition: Schema's `context_fields` section lists all three fields with types and required=true.
  - Error: If a required field is missing from the handoff, the Orchestrator SHALL halt with: "Missing required field '<name>' in reviewer-to-orchestrator handoff."

- **FR-023**: A schema file SHALL exist at `.github/schemas/coder-complete-to-orchestrator.schema.yaml` defining the Coder's completion signal to the Orchestrator.
  - Precondition: `.github/schemas/` directory exists.
  - Postcondition: Schema file exists and follows the handoff/v1 format.
  - Error: If the file is missing, the Orchestrator's schema validation step SHALL report it.

- **FR-024**: The coder-complete-to-orchestrator schema SHALL require: `wp_path` (string, required) and `lane_confirmation` (literal value `for_review`, required).
  - Precondition: Schema file exists per FR-023.
  - Postcondition: Schema's `context_fields` section lists both fields.
  - Error: If `lane_confirmation` is not `for_review`, the Orchestrator SHALL halt with: "Coder completion handoff has invalid lane_confirmation. Expected 'for_review'."

- **FR-025**: A schema file SHALL exist at `.github/schemas/docs-agent-to-orchestrator.schema.yaml` defining the Docs Agent's completion signal to the Orchestrator.
  - Precondition: `.github/schemas/` directory exists.
  - Postcondition: Schema file exists and follows the handoff/v1 format.
  - Error: If the file is missing, the Orchestrator's schema validation step SHALL report it.

- **FR-026**: The docs-agent-to-orchestrator schema SHALL require: `wp_path` (string, required) and `docs_completed` (boolean, required).
  - Precondition: Schema file exists per FR-025.
  - Postcondition: Schema's `context_fields` section lists both fields.
  - Error: If `docs_completed` is not true, the Orchestrator SHALL log a warning and treat documentation as incomplete.

- **FR-027**: All three return handoff schemas SHALL follow the existing `handoff/v1` format with `schema`, `source_agent`, `target_agent`, `description`, `required_artifacts`, `required_state`, `context_fields`, and `validation_rules` sections.
  - Precondition: Existing handoff schemas exist as format reference.
  - Postcondition: Return schemas are structurally consistent with forward schemas.
  - Error: If a required schema section is missing, the validating agent SHALL halt listing the missing sections.

#### Implementation Contract -- Return Handoff Schemas

**Inputs**: None (new file creation following handoff/v1 template)
**Outputs**: Three new schema files in `.github/schemas/`
**Error behaviors**:
- Schema file missing -> Orchestrator halt with descriptive error
- Required context field missing from handoff -> Orchestrator halt with field name
- Unexpected field value -> Orchestrator halt with expected vs actual values

**Files created**:
- `.github/schemas/reviewer-to-orchestrator.schema.yaml`
- `.github/schemas/coder-complete-to-orchestrator.schema.yaml`
- `.github/schemas/docs-agent-to-orchestrator.schema.yaml`

---

### 4.5 Error-Handling Policy Documentation (C5)

- **FR-028**: The architecture documentation at `.sdd/docs/architecture.md` SHALL contain a subsection titled "Design Decision: Error-Handling Policy" within its Design Decisions section.
  - Precondition: `.sdd/docs/architecture.md` exists with a Design Decisions section.
  - Postcondition: The subsection exists and documents the error-handling asymmetry.
  - Error: If architecture.md does not exist or lacks a Design Decisions section, the implementer SHALL create the section.

- **FR-029**: The error-handling policy documentation SHALL explicitly categorize each agent as either critical-path (HALT on failure) or advisory (best-effort on failure):
  - Critical-path: Spec Architect, Planner, Coder -- errors compound downstream, so these agents HALT on skill failure.
  - Advisory: Review Coordinator, Docs Agent -- partial output is still valuable, so these agents continue to the next skill on failure.
  - Precondition: Architecture documentation subsection exists per FR-028.
  - Postcondition: Every pipeline agent is categorized with rationale.
  - Error: If a new agent is added without categorization, the review SHALL flag it.

- **FR-030**: Each agent file (`.github/agents/*.agent.md`) SHALL include a one-line reference to the error-handling policy: `<!-- Error policy: See .sdd/docs/architecture.md, Design Decision: Error-Handling Policy -->`.
  - Precondition: Agent file exists.
  - Postcondition: Agent file contains the reference comment.
  - Error: Missing reference is a spec compliance issue, not a runtime error.

#### Implementation Contract -- Error-Handling Policy

**Inputs**: Agent categorization table (critical-path vs advisory)
**Outputs**: Architecture doc subsection; agent file reference comments
**Error behaviors**:
- Architecture doc missing -> create the Design Decisions section
- Agent file missing reference -> spec compliance violation (non-blocking)

**Files modified**:
- `.sdd/docs/architecture.md` -- add Design Decision subsection
- All 8 `.github/agents/*.agent.md` files -- add policy reference comment

---

### 4.6 Acceptance Criteria RACI (C6)

- **FR-031**: The Coder agent instructions SHALL explicitly document the Coder's role as "Responsible" for checking off acceptance criteria checkboxes (`- [ ]` to `- [x]`) during implementation.
  - Precondition: Coder agent instructions discuss acceptance criteria.
  - Postcondition: The role is labeled "Responsible (maker)" in the agent instructions.
  - Error: If the existing instructions already describe this role, add the explicit "Responsible" label without changing behavior.

- **FR-032**: The Review Coordinator agent instructions SHALL explicitly document its role as "Accountable/Verifier" for confirming that checked-off acceptance criteria match actual implementation.
  - Precondition: Review Coordinator agent instructions discuss acceptance criteria verification.
  - Postcondition: The role is labeled "Accountable/Verifier (checker)" in the agent instructions.
  - Error: Same as FR-031.

- **FR-033**: The developer guide (`.sdd/docs/developer-guide.md`) SHALL document the maker/checker pattern: the Coder checks boxes, the Reviewer verifies them, and this is intentional dual-touch, not redundancy.
  - Precondition: Developer guide exists.
  - Postcondition: Developer guide contains an "Acceptance Criteria Ownership" section.
  - Error: If developer guide does not exist, the section SHALL be created in a new file.

#### Implementation Contract -- Acceptance Criteria RACI

**Inputs**: RACI assignments: Coder=Responsible, Reviewer=Accountable
**Outputs**: Updated agent instructions and developer guide section
**Error behaviors**:
- Existing role description conflicts with RACI -> update to match RACI, preserve existing behavior
- Developer guide missing -> create the relevant section

**Files modified**:
- `.github/agents/coder.agent.md` -- add "Responsible (maker)" label
- `.github/agents/review-coordinator.agent.md` -- add "Accountable/Verifier (checker)" label
- `.sdd/docs/developer-guide.md` -- add maker/checker documentation

---

### 4.7 Configurable Coverage Thresholds (C7)

- **FR-034**: WP file YAML frontmatter SHALL support an optional `coverage_code` field of type integer (range 0-100) representing the minimum code coverage percentage.
  - Precondition: WP file exists with valid YAML frontmatter.
  - Postcondition: WP frontmatter schema accepts `coverage_code` as a valid optional field.
  - Error: If `coverage_code` is present but out of range (< 0 or > 100) or not an integer, the reading skill SHALL halt with: "Invalid coverage_code value '<value>'. Must be an integer 0-100."

- **FR-035**: WP file YAML frontmatter SHALL support an optional `coverage_branch` field of type integer (range 0-100) representing the minimum branch coverage percentage.
  - Precondition: WP file exists with valid YAML frontmatter.
  - Postcondition: WP frontmatter schema accepts `coverage_branch` as a valid optional field.
  - Error: Same validation as FR-034 but for `coverage_branch`.

- **FR-036**: The code-unit-tests skill SHALL read `coverage_code` and `coverage_branch` from WP frontmatter and use them as the minimum coverage thresholds for the test run.
  - Precondition: code-unit-tests skill is dispatched for a WP.
  - Postcondition: Coverage thresholds used for enforcement match WP frontmatter values.
  - Error: If frontmatter values cannot be read, fall back to defaults per FR-037.

- **FR-037**: The code-unit-tests skill SHALL fall back to default thresholds of 80% code coverage and 90% branch coverage when `coverage_code` or `coverage_branch` frontmatter fields are absent.
  - Precondition: WP frontmatter does not contain `coverage_code` or `coverage_branch`.
  - Postcondition: Skill uses 80% and 90% as thresholds.
  - Error: None -- this is the default behavior path.

- **FR-038**: The code-env-setup skill SHALL read and apply `coverage_code` and `coverage_branch` from WP frontmatter when configuring coverage tooling, with the same fallback defaults as FR-037.
  - Precondition: code-env-setup skill is configuring test tooling.
  - Postcondition: Coverage tool configuration uses WP-specific or default thresholds.
  - Error: Same as FR-037.

- **FR-039**: The spec-test-strategy skill SHALL state that coverage thresholds are configurable per-WP via `coverage_code` and `coverage_branch` frontmatter fields, with defaults of 80% code and 90% branch.
  - Precondition: spec-test-strategy skill is writing Section 11 of a spec.
  - Postcondition: Section 11 references configurable thresholds with defaults.
  - Error: If hardcoded values appear instead of configurable references, review SHALL flag it.

#### Implementation Contract -- Configurable Coverage Thresholds

**Inputs**: WP frontmatter fields `coverage_code` (optional integer 0-100), `coverage_branch` (optional integer 0-100)
**Outputs**: Coverage threshold values (integer) used by test tooling
**Error behaviors**:
- Field absent -> use default (80% code, 90% branch)
- Field present but invalid type -> halt with descriptive error
- Field present but out of range -> halt with range error
- WP file unreadable -> halt (critical skill failure)

**Files modified**:
- `.github/skills/code-unit-tests/SKILL.md` -- add frontmatter threshold reading with fallback
- `.github/skills/code-env-setup/SKILL.md` -- add frontmatter threshold reading with fallback
- `.github/skills/spec-test-strategy/SKILL.md` -- update to reference configurable thresholds

---

### 4.8 Dependency-Aware WP Ordering (C8)

- **FR-040**: The Orchestrator SHALL select the next WP for implementation using a topological sort of the dependency graph derived from `depends_on` frontmatter fields in WP files.
  - Precondition: Orchestrator is selecting the next WP. WP files exist with optional `depends_on` fields.
  - Postcondition: The selected WP has all dependencies in `lane: done` (or has no dependencies).
  - Error: If no WP has all dependencies met and at least one WP has unmet dependencies with all dependency WPs not in `lane: done`, the Orchestrator SHALL report: "No WPs are ready. Blocked WPs: <list with unmet deps>."

- **FR-041**: When multiple WPs have no unmet dependencies and are eligible for implementation (lane=planned), the Orchestrator SHALL use the lowest WP number as tiebreaker.
  - Precondition: Topological sort produces multiple WPs at the same priority level.
  - Postcondition: The lowest-numbered WP among equally-eligible WPs is selected.
  - Error: None -- tiebreaker is deterministic.

- **FR-042**: The Orchestrator SHALL detect circular dependencies in the dependency graph and halt with an error message listing the cycle.
  - Precondition: Orchestrator builds the dependency graph from WP `depends_on` fields.
  - Postcondition: If a cycle exists, the Orchestrator halts before selecting any WP.
  - Error: "Circular dependency detected: <WP-A> -> <WP-B> -> ... -> <WP-A>. Cannot determine execution order."

- **FR-043**: WPs with no `depends_on` field or an empty `depends_on` field SHALL be treated as having no dependencies (eligible immediately).
  - Precondition: WP frontmatter has no `depends_on` key or `depends_on: []`.
  - Postcondition: WP is eligible for implementation as soon as its lane is `planned`.
  - Error: None -- this is the default case.

#### Implementation Contract -- Dependency-Aware WP Ordering

**Inputs**: List of WP files with frontmatter fields: `lane` (string), `depends_on` (optional array of WP identifiers)
**Outputs**: Single WP identifier representing the next WP to implement
**Error behaviors**:
- Circular dependency -> halt with cycle description
- All remaining WPs blocked by unmet dependencies -> report blocked status
- `depends_on` references non-existent WP -> halt with: "WP<NN> depends on <dep> which does not exist"
- No WPs with lane=planned -> no action needed

**Files modified**:
- `.github/agents/orchestrator.agent.md` -- replace "WP Selection Priority" section with topological sort algorithm

---

### 4.9 Schema Versioning Protocol (C9)

- **FR-044**: Each handoff schema file in `.github/schemas/` SHALL include a `version_history` section at the end of the file documenting all schema changes.
  - Precondition: Schema file exists.
  - Postcondition: Schema file contains a `version_history` YAML array with at least one entry (the initial version).
  - Error: If `version_history` is missing, the schema validation step SHALL log a warning but not halt.

- **FR-045**: A version history entry SHALL contain: `version` (string, e.g., "handoff/v1"), `date` (ISO-8601 date string), and `description` (string summarizing changes).
  - Precondition: Schema file contains `version_history` section.
  - Postcondition: Each entry has all three fields.
  - Error: If an entry is missing a field, the schema validation step SHALL log a warning.

- **FR-046**: Breaking changes to a schema (removing required fields, changing field types, removing enum values, renaming fields) SHALL increment the schema version from `handoff/vN` to `handoff/v(N+1)`.
  - Precondition: A schema modification has been identified as breaking.
  - Postcondition: `schema` field is updated to the next version and a version_history entry is added.
  - Error: If a breaking change is made without version increment, the review SHALL flag it.

- **FR-047**: Additive changes (new optional fields, new enum values, new optional validation rules) SHALL retain the current schema version number.
  - Precondition: A schema modification has been identified as additive.
  - Postcondition: `schema` field remains unchanged; a version_history entry notes the addition.
  - Error: None -- this is a convention, not a runtime check.

- **FR-048**: Each schema SHALL validate artifact path placeholders using these regex patterns: `{NNN}` matches `\d{2,3}`, `{name}` matches `[a-z0-9-]+`, `{slug}` matches `[a-z0-9-]+`, `{NN}` matches `\d{2}`.
  - Precondition: Schema contains path patterns with placeholders.
  - Postcondition: Each placeholder has an accompanying YAML comment with the regex pattern.
  - Error: If a placeholder does not match its regex during validation, the agent SHALL halt with: "Artifact path '<path>' does not match expected pattern for placeholder '<placeholder>'."

#### Implementation Contract -- Schema Versioning Protocol

**Inputs**: Schema file path; change type (breaking or additive)
**Outputs**: Updated schema file with version_history entry and optionally incremented version
**Error behaviors**:
- Missing version_history -> warning (non-blocking)
- Breaking change without version bump -> review finding (non-blocking)
- Path placeholder regex mismatch -> halt with descriptive error

**Files modified**:
- All `.github/schemas/*.schema.yaml` files -- add version_history sections
- A new "Schema Versioning Protocol" section in `.sdd/docs/developer-guide.md`

---

### 4.10 Shared Schema Base (C10)

- **FR-049**: A shared base schema SHALL exist at `.github/schemas/base-handoff.schema.yaml`.
  - Precondition: `.github/schemas/` directory exists.
  - Postcondition: Base schema file exists and defines reusable validation patterns.
  - Error: If the base schema is missing, individual schemas that reference it SHALL fall back to inline validation rules and log a warning.

- **FR-050**: The base schema SHALL define these reusable validation patterns: WP file existence check (file at path exists and has valid YAML frontmatter), lane field validation (field value is one of the `lane` enum values), and file path format validation (path matches expected pattern).
  - Precondition: base-handoff.schema.yaml exists.
  - Postcondition: Three named validation patterns are defined in the file.
  - Error: If a referenced pattern name is not found in the base schema, the agent SHALL halt with: "Base schema pattern '<name>' not found."

- **FR-051**: Individual handoff schemas SHALL reference the base schema using a `base_schema` field pointing to `.github/schemas/base-handoff.schema.yaml` and use `$ref`-style references for common validation rules.
  - Precondition: Base schema exists with defined patterns.
  - Postcondition: At least two individual schemas reference the base schema instead of duplicating validation rules.
  - Error: If the referenced base schema cannot be loaded, the individual schema's inline rules (if any) SHALL be used as fallback.

#### Implementation Contract -- Shared Schema Base

**Inputs**: None (new file creation)
**Outputs**: `.github/schemas/base-handoff.schema.yaml` with reusable validation patterns
**Error behaviors**:
- Base schema missing -> individual schemas use inline rules (degraded mode)
- Referenced pattern not found -> halt with pattern name
- Base schema YAML parse error -> halt with parse error details

**Files created**:
- `.github/schemas/base-handoff.schema.yaml`

**Files modified**:
- At least 2 existing `.github/schemas/*.schema.yaml` files -- add `base_schema` reference

---

### 4.11 Pattern File Propagation (C11)

- **FR-052**: Each domain patterns file (`.sdd/reviews/spec-patterns.md`, `.sdd/reviews/plan-patterns.md`, `.sdd/reviews/code-patterns.md`, `.sdd/reviews/doc-patterns.md`) SHALL include a `patterns_version` field in its YAML frontmatter, initialized to 1.
  - Precondition: Patterns file exists.
  - Postcondition: Patterns file has `patterns_version: <integer>` in its YAML frontmatter.
  - Error: If frontmatter is missing, the coordinator SHALL treat `patterns_version` as 0 (always reload).

- **FR-053**: The Review Coordinator SHALL increment `patterns_version` by 1 each time it adds, modifies, or retires a pattern in a domain patterns file.
  - Precondition: Review Coordinator is updating a patterns file.
  - Postcondition: `patterns_version` in the file's frontmatter is incremented by 1.
  - Error: If `patterns_version` is missing, the Review Coordinator SHALL add it with value 1.

- **FR-054**: Coordinator agents (Spec Architect, Planner, Coder, Docs Agent) SHALL record the `patterns_version` value when they first read the patterns file and SHALL re-read the file before each skill dispatch if `patterns_version` has changed.
  - Precondition: Coordinator is about to dispatch a skill subagent.
  - Postcondition: Skill receives the most recent patterns content.
  - Error: If the patterns file is unreadable on re-check, the coordinator SHALL use the last successfully read patterns and log a warning.

#### Implementation Contract -- Pattern File Propagation

**Inputs**: Patterns file path (string); previously read `patterns_version` (integer)
**Outputs**: Current patterns content (string); updated `patterns_version` (integer)
**Error behaviors**:
- Patterns file missing frontmatter -> treat version as 0, always reload
- Patterns file unreadable on re-check -> use cached patterns, log warning
- Version not an integer -> treat as 0, always reload

**Files modified**:
- `.sdd/reviews/spec-patterns.md` -- add `patterns_version` frontmatter
- `.sdd/reviews/plan-patterns.md` -- add `patterns_version` frontmatter
- `.sdd/reviews/code-patterns.md` -- add `patterns_version` frontmatter
- `.sdd/reviews/doc-patterns.md` -- add `patterns_version` frontmatter
- `.github/agents/spec-architect.agent.md` -- add version-check-before-dispatch logic
- `.github/agents/planner.agent.md` -- add version-check-before-dispatch logic
- `.github/agents/coder.agent.md` -- add version-check-before-dispatch logic
- `.github/agents/docs-agent.agent.md` -- add version-check-before-dispatch logic
- `.github/agents/review-coordinator.agent.md` -- add patterns_version increment logic

---

### 4.12 Contract File Validation Pilot (C12)

- **FR-055**: An integration test document SHALL exist at `.sdd/tests/contract-validation-pilot.md` describing a test scenario that exercises the Coder agent against a small, real codebase (not markdown/YAML pipeline files).
  - Precondition: The pipeline is functional and has been used to build itself.
  - Postcondition: Test document exists with scenario description, expected outcomes, and actual results.
  - Error: If the test cannot be executed (no suitable codebase available), the document SHALL record this as a known limitation.

- **FR-056**: The test scenario SHALL exercise three Coder capabilities: contract-first implementation (reading contract files and implementing to match), coverage threshold enforcement (verifying tests meet thresholds), and debug retry loops (handling test failures and retrying).
  - Precondition: Test document exists per FR-055.
  - Postcondition: Each of the three capabilities has a dedicated test case with pass/fail criteria.
  - Error: If a capability cannot be tested, the document SHALL record why and propose an alternative validation approach.

- **FR-057**: The test document SHALL record results and any discovered issues, with each issue categorized as: blocking (prevents real-world use), degraded (works but suboptimally), or cosmetic (minor).
  - Precondition: Test scenario has been executed.
  - Postcondition: Results section contains categorized findings.
  - Error: If the test was not executed, the results section SHALL state "Not yet executed" with a reason.

#### Implementation Contract -- Contract File Validation Pilot

**Inputs**: Test scenario definition; coder agent; small real codebase
**Outputs**: `.sdd/tests/contract-validation-pilot.md` with scenario, results, and findings
**Error behaviors**:
- No suitable test codebase -> document limitation, defer execution
- Coder agent fails during test -> record as blocking finding
- Partial success -> categorize each finding by severity

**Files created**:
- `.sdd/tests/contract-validation-pilot.md`

---

## 10. Non-Functional Requirements

### 10.1 Performance

- **NFR-001**: Adding the enum registry file and return handoff schemas SHALL NOT increase agent initialization time perceptibly. The total additional file reads per agent startup SHALL be no more than 3 files (enums.yaml, base-handoff.schema.yaml, and one return schema).
  - Measurement: Count of additional file reads per agent invocation.

- **NFR-002**: The Orchestrator's topological sort for WP ordering SHALL complete in O(V+E) time where V is the number of WPs and E is the number of dependency edges, supporting up to 100 WPs without perceptible delay.
  - Measurement: Algorithmic complexity analysis; manual verification for projects with > 20 WPs.

### 10.2 Security

This system is a file-based LLM agent pipeline with no HTTP server, no database, no user authentication, and no network-facing attack surface. Security requirements focus on data integrity, secret avoidance, and safe file operations.

#### 10.2.1 Authentication & Authorization

Not applicable. The pipeline has no runtime authentication. Access control is managed by Git repository permissions and VS Code Copilot Chat session context.

#### 10.2.2 Data Classification

| Entity | Field | Classification | Handling |
|--------|-------|---------------|----------|
| WP Frontmatter | lane, review_cycles, docs_completed | internal | No special handling; version-controlled |
| WP Frontmatter | coverage_code, coverage_branch | internal | No special handling |
| Enum Registry | All enum values | public | Intentionally discoverable; no sensitive data |
| Handoff Schema | context_fields | internal | SHALL NOT contain secrets, tokens, or credentials |
| Activity Log | details field | internal | SHALL NOT contain secrets, stack traces, or PII |
| Pattern Files | patterns_version, pattern content | internal | No sensitive data |
| Pipeline State | error_log entries | internal | error_summary SHALL NOT contain full stack traces |

- **NFR-003**: No file created or modified by this spec SHALL contain secrets, API keys, tokens, or credentials.

- **NFR-004**: No new file permissions or access patterns are introduced beyond what existing schema files require. All new files follow the same read-only-by-agents pattern.

#### 10.2.3 OWASP Top 10 Assessment

| # | OWASP Category | Applies | Assessment |
|---|---------------|---------|------------|
| A01 | Broken Access Control | No | No runtime access control. File access governed by Git/OS permissions. |
| A02 | Cryptographic Failures | No | No encryption or cryptographic operations. |
| A03 | Injection | Partial | Malformed YAML causes parse errors, not code injection. Mitigation: agents halt on parse errors. |
| A04 | Insecure Design | Yes | This spec replaces implicit conventions with explicit contracts. |
| A05 | Security Misconfiguration | Partial | Missing registry/schema files cause halts. Mitigation: explicit error messages. |
| A06 | Vulnerable Components | No | No third-party dependencies. All files are first-party. |
| A07 | Auth Failures | No | No authentication system. |
| A08 | Data Integrity Failures | Yes | Registry/schema tampering mitigated by Git version control and review. |
| A09 | Logging Failures | Partial | Inconsistent log formats addressed by C3 canonical format. |
| A10 | SSRF | No | No network requests. All operations are local file reads/writes. |

#### 10.2.4 Input Validation

- Agents SHALL validate enum values against enums.yaml before state transitions.
- Agents SHALL validate file paths against expected patterns before reading or writing.
- Agents SHALL validate YAML frontmatter structure before extracting field values.
- Return handoff schemas SHALL validate file paths match `^\.sdd/plans/WP\d{2}-[a-z0-9-]+\.md$`.
- Activity Log entries SHALL NOT expose filesystem paths outside the workspace root.

### 10.3 Scalability

- **NFR-005**: The central enum registry SHALL support the addition of new enum groups and new values within existing groups without requiring changes to consuming agents (agents read the file dynamically, not at compile time).

- **NFR-006**: The shared base schema pattern SHALL support up to 20 individual handoff schemas referencing it without structural changes to the base schema format.

### 10.4 Accessibility

- **NFR-007**: All new documentation (error-handling policy, schema versioning protocol, RACI documentation) SHALL use plain language readable at a professional technical level, with no jargon not defined in the spec glossary.

### 10.5 Observability

- **NFR-008**: The standardized Activity Log format SHALL be parseable by simple text tools (grep, awk) for debugging, with each field separated by ` - ` (space-hyphen-space).

- **NFR-009**: When an agent reads a frontmatter field with an unexpected type or missing value, it SHALL log a warning message including the file path, field name, expected type, and actual value before applying the default.

### 10.6 Backward Compatibility

- **NFR-010**: All changes SHALL be backward-compatible with existing WP files. WP files created before this hardening pass (lacking `review_cycles`, `docs_completed`, `coverage_code`, `coverage_branch` fields) SHALL be processed without errors by all agents.

- **NFR-011**: Existing handoff schemas SHALL continue to function without modification. New schemas are additive and do not require version bumps on existing schemas (unless the base schema reference is added, which is an optional additive field).

### 10.7 Maintainability

- **NFR-012**: The enum registry SHALL be the single source of truth for all enum values. No agent or schema file SHALL define enum values inline without a reference comment pointing to enums.yaml.

- **NFR-013**: Schema versioning rules SHALL be documented in the developer guide so that future schema changes follow a consistent protocol without requiring this spec as reference.

---

## 12. Constraints & Assumptions

### 12.1 Constraints

| # | Constraint | Impact |
|---|-----------|--------|
| C-01 | Deliverables are markdown (.agent.md, SKILL.md) and YAML (.schema.yaml) files, not executable code | Implementation is text editing, not software compilation. Testing is manual review and pipeline execution, not unit tests. |
| C-02 | The existing agent/skill architecture is preserved -- no agents added, removed, or renamed | All changes are modifications to existing agent instructions or additions of new schema/config files |
| C-03 | YAML is the only schema format (no JSON Schema, OpenAPI, or other conversions) | Schemas remain human-readable but lack automated programmatic validation tooling |
| C-04 | The pipeline remains sequential -- no parallel WP execution | C8 (dependency ordering) improves ordering correctness but does not enable parallelism |
| C-05 | Agent files are interpreted by LLM agents, not by compilers or interpreters | "Validation" means the agent reads and follows instructions, not programmatic enforcement |
| C-06 | The pipeline has built itself through 9 specs and 39 WPs, so all changes must be validated against the existing self-referential build history | Regression risk requires careful backward compatibility |

### 12.2 Assumptions

| # | Assumption | If wrong |
|---|-----------|----------|
| A-01 | Adding frontmatter fields to WP files is backward-compatible because agents check for field presence before reading | Agents could fail on older WP files. Mitigation: FR-007 explicitly requires default-on-absent behavior. |
| A-02 | A single enums.yaml file is sufficient -- no agent needs runtime-generated or dynamically computed enum values | The registry becomes stale if an agent creates ad-hoc values. Mitigation: enum values change infrequently. |
| A-03 | The existing handoff/v1 schema format is sufficient for return handoff schemas | Return handoffs may need different structure. Mitigation: handoff/v1 is flexible (arbitrary context_fields). |
| A-04 | Standardizing Activity Log format does not break any tooling outside the pipeline | External log parsers could break. Mitigation: no external consumers identified in codebase assessment. |
| A-05 | Coverage threshold overrides in WP frontmatter will be correctly propagated to test runners by skills | Skills may have hardcoded paths that ignore frontmatter. Mitigation: FR-036/FR-037 explicitly require frontmatter reading with fallback. |
| A-06 | The `depends_on` field in WP frontmatter contains valid WP identifiers that correspond to existing WP files | Invalid references could cause the topological sort to fail. Mitigation: FR-042/FR-043 include validation. |

---

## 13. Out of Scope

- **Full pipeline redesign**: This is a hardening pass, not a rewrite. Agent architecture, skill dispatch patterns, and the overall pipeline flow remain unchanged. Rationale: minimize risk and scope.
- **Runtime validation tooling**: No new CLI tools, linters, or automated schema validators are created. Validation remains agent-driven (agents read schemas and check conditions in their instructions). Rationale: the pipeline has no executable runtime; agents are the "runtime."
- **Multi-language schema formats**: Schemas stay in YAML. No JSON Schema, OpenAPI, or other format conversions are produced. Rationale: consistency with existing infrastructure.
- **Parallel WP execution**: The pipeline remains sequential even with dependency-aware ordering. Rationale: agents are single-threaded LLM conversations; parallelism requires architectural changes.
- **Pattern file real-time push**: No event system or file-watching mechanism. C11 uses a poll-on-dispatch approach where the coordinator checks the version counter before each skill dispatch. Rationale: simplicity; push-based systems require infrastructure not present in the pipeline.
- **New agent creation**: No new agents are added to the pipeline. All capabilities are implemented through modifications to existing agent and skill files, plus new schema/config files. Rationale: preserve the existing 8-agent architecture.
- **Automated enum validation**: No tooling enforces that agents use only registry-defined enum values. Enforcement is via agent instructions and review. Rationale: no executable validation layer exists.
- **Historical WP migration**: Existing WP files from previous pipeline runs are not retroactively updated with new frontmatter fields. New fields are only written going forward. Rationale: backward compatibility via default-on-absent (FR-007).

---

## 5. User Stories

### Pipeline Operator Stories

### US-01 -- Reliable Review Cycle Tracking (Priority: P1) MVP

**As a** Pipeline Operator, **I want** the Orchestrator to read review cycle counts from WP frontmatter fields, **so that** escalation decisions are based on reliable structured data instead of fragile Activity Log text parsing.

**Why P1**: Review cycle miscounts can trigger false escalations (wasting user time) or missed escalations (allowing infinite rework loops). This is the highest-impact fragility in the current pipeline.

**Independent Test**: Create a WP file with `review_cycles: 3` in frontmatter and verify the Orchestrator triggers escalation without any Activity Log entries.

**Acceptance Scenarios**:
1. **Given** a WP with `review_cycles: 2` in frontmatter and the Review Coordinator returns verdict "Changes Required", **When** the Review Coordinator updates the WP, **Then** `review_cycles` is incremented to 3 and the Orchestrator escalates to the user instead of re-invoking the Coder. (FR-002, FR-005)
2. **Given** a WP file created before this hardening pass (no `review_cycles` field), **When** the Orchestrator evaluates it for escalation, **Then** it treats the missing field as 0 and does not escalate. (FR-007)
3. **Given** a WP with `review_cycles: "invalid"` (non-integer), **When** an agent reads it, **Then** the agent treats it as 0 and logs a warning. (FR-001)

### Edge Cases
- What happens if `review_cycles` is negative? The agent treats it as 0 and logs a warning (same as non-integer handling).
- What happens if the Review Coordinator fails mid-write (incremented review_cycles but did not change lane)? The next invocation will read the already-incremented value; the Orchestrator cross-verifies frontmatter state on startup (existing behavior).

---

### US-02 -- Reliable Documentation Completion Tracking (Priority: P1) MVP

**As a** Pipeline Operator, **I want** the Orchestrator to read documentation completion status from WP frontmatter, **so that** it correctly determines which WPs still need documentation without scanning Activity Logs.

**Why P1**: Incorrect docs tracking causes the Orchestrator to either skip documentation (WP appears documented when it is not) or re-invoke the Docs Agent unnecessarily.

**Independent Test**: Create a WP file with `lane: done` and `docs_completed: true`, verify the Orchestrator advances to the next WP instead of invoking the Docs Agent.

**Acceptance Scenarios**:
1. **Given** a WP with `lane: done` and `docs_completed: false`, **When** the Orchestrator evaluates next action, **Then** it invokes the Docs Agent for that WP. (FR-006)
2. **Given** a WP with `lane: done` and no `docs_completed` field, **When** the Orchestrator evaluates next action, **Then** it treats the WP as not yet documented (default false) and invokes the Docs Agent. (FR-007)
3. **Given** the Docs Agent completes successfully for a WP, **When** it finishes, **Then** it sets `docs_completed: true` in the WP frontmatter. (FR-004)

### Edge Cases
- What happens if the Docs Agent fails partway through (some but not all doc skills completed)? The Docs Agent uses best-effort; `docs_completed` is still set to true because the agent completed its invocation. Individual skill failures are recorded in the commit.
- What happens if `docs_completed` is a string "true" instead of boolean true? The agent treats non-boolean values as false and logs a warning (FR-003).

---

### US-03 -- Clear Error Policy Understanding (Priority: P1) MVP

**As a** Pipeline Operator, **I want** the error-handling policy to be explicitly documented, **so that** I understand why some agent failures halt the pipeline while others are silently tolerated.

**Why P1**: Users are confused when security review skills fail silently (best-effort) but a Coder skill failure halts everything. Without documented rationale, users lose trust in the pipeline.

**Independent Test**: Read `.sdd/docs/architecture.md` and verify it contains a "Design Decision: Error-Handling Policy" section categorizing every agent.

**Acceptance Scenarios**:
1. **Given** a user reads `.sdd/docs/architecture.md`, **When** they look for error-handling guidance, **Then** they find a "Design Decision: Error-Handling Policy" subsection listing every agent as either "critical-path (HALT)" or "advisory (best-effort)" with rationale. (FR-028, FR-029)
2. **Given** a user reads any agent file, **When** they look for error policy, **Then** they find a reference comment pointing to the architecture doc. (FR-030)

### Edge Cases
- What if a new agent is added in the future without categorization? The review process SHALL flag the missing categorization. The architecture doc remains the authoritative source.

---

### US-04 -- Configurable Coverage Thresholds (Priority: P2)

**As a** Pipeline Operator, **I want** to override coverage thresholds per-WP via frontmatter fields, **so that** I can set project-appropriate thresholds without modifying skill files.

**Why P2**: Hardcoded thresholds (80%/90%) are reasonable defaults but inappropriate for all projects. Some projects need higher coverage for safety-critical code; others accept lower coverage for prototypes.

**Independent Test**: Create a WP with `coverage_code: 60` and `coverage_branch: 70` in frontmatter, run the Coder, and verify the coverage tool enforces 60%/70% instead of 80%/90%.

**Acceptance Scenarios**:
1. **Given** a WP with `coverage_code: 60` and `coverage_branch: 70`, **When** the code-unit-tests skill runs, **Then** it enforces 60% code and 70% branch coverage thresholds. (FR-036)
2. **Given** a WP with no coverage fields in frontmatter, **When** the code-unit-tests skill runs, **Then** it enforces the defaults: 80% code, 90% branch. (FR-037)
3. **Given** a WP with `coverage_code: 150` (out of range), **When** the code-unit-tests skill reads it, **Then** it halts with: "Invalid coverage_code value '150'. Must be an integer 0-100." (FR-034)

### Edge Cases
- What if only `coverage_code` is specified but not `coverage_branch`? Each field is independent; the specified one uses the override, the absent one uses the default.
- What if coverage_code is 0? It is valid (no code coverage required). Some prototyping WPs may legitimately set this.

---

### US-05 -- Correct WP Execution Order (Priority: P2)

**As a** Pipeline Operator, **I want** the Orchestrator to select WPs based on dependency graph topology, **so that** WPs with dependencies are never started before their prerequisites are complete.

**Why P2**: The current lowest-number-first approach works when WP numbering aligns with dependency order, but breaks when the Planner creates dependencies that do not follow numbering (e.g., WP10 depends on WP15).

**Independent Test**: Create WP01 (depends_on: [WP02]) and WP02 (no deps), verify the Orchestrator selects WP02 first despite WP01 having a lower number.

**Acceptance Scenarios**:
1. **Given** WP01 depends on WP02 and both have `lane: planned`, **When** the Orchestrator selects the next WP, **Then** it selects WP02 (dependency-free) first. (FR-040, FR-041)
2. **Given** WP01 depends on WP02 and WP02 depends on WP01 (circular), **When** the Orchestrator builds the dependency graph, **Then** it halts with: "Circular dependency detected: WP01 -> WP02 -> WP01." (FR-042)
3. **Given** a WP with `depends_on: [WP99]` where WP99 does not exist, **When** the Orchestrator builds the dependency graph, **Then** it halts with: "WP01 depends on WP99 which does not exist." (FR-042)

### Edge Cases
- What if all WPs have no dependencies? The sort degenerates to lowest-number-first (same as current behavior).
- What if a dependency WP has `lane: blocked`? The dependent WP cannot proceed; the Orchestrator reports it as blocked with the reason.

---

### Pipeline Maintainer Stories

### US-06 -- Central Enum Registry (Priority: P1) MVP

**As a** Pipeline Maintainer, **I want** all enum values (lane, spec_status, pipeline_stage, review_status) defined in a single registry file, **so that** I can add or modify values in one place instead of hunting through multiple agent files.

**Why P1**: Without a central registry, enum values are implicitly defined by scattered references. Adding a new lane value requires finding and updating every agent and schema that uses lanes -- error-prone and untraceable.

**Independent Test**: Read `.github/schemas/enums.yaml` and verify it contains all four enum groups with the canonical values. Search all agent files for enum reference comments.

**Acceptance Scenarios**:
1. **Given** a maintainer reads `.github/schemas/enums.yaml`, **When** they look up valid lane values, **Then** they find exactly: planned, doing, for_review, to_do, done, blocked. (FR-010)
2. **Given** a maintainer searches for "Final" in the Planner agent file, **When** they check spec_status references, **Then** no "Final" references exist -- only Draft, Validated, Approved per the registry. (FR-015)
3. **Given** enums.yaml is missing from the repository, **When** an agent starts and tries to reference it, **Then** the agent halts with: "Enum registry not found at .github/schemas/enums.yaml." (FR-008)

### Edge Cases
- What if a maintainer adds a new lane value to enums.yaml but does not update agent instructions? The value is valid per the registry but agents may not know how to handle it. The review process should catch behavioral gaps.
- What if enums.yaml has a YAML syntax error? Agents halt with a parse error. The file should be validated before committing.

---

### US-07 -- Consistent Activity Log Format (Priority: P1) MVP

**As a** Pipeline Maintainer, **I want** all agents to use the same Activity Log entry format, **so that** logs are consistently readable and I can search them with standard text tools.

**Why P1**: Inconsistent formats make Activity Logs unreliable for debugging. Different agents use slightly different separators and field orders, making grep-based investigation fragile.

**Independent Test**: Run a full pipeline cycle and verify every Activity Log entry in every WP file matches the canonical format.

**Acceptance Scenarios**:
1. **Given** the Coder appends an Activity Log entry, **When** the entry is written, **Then** it follows: `<ISO-8601> - coder - lane=doing - Starting implementation`. (FR-016, FR-017)
2. **Given** the Review Coordinator appends an Activity Log entry, **When** the entry is written, **Then** it follows the same canonical format. (FR-018)
3. **Given** the canonical format is documented in enums.yaml conventions section, **When** a maintainer looks up the format, **Then** they find a single authoritative definition. (FR-020)

### Edge Cases
- What if an agent writes a log entry with a malformed timestamp? The entry is written as-is (no runtime validation of log format), but review should catch format violations.

---

### US-08 -- Complete Handoff Schema Coverage (Priority: P1) MVP

**As a** Pipeline Maintainer, **I want** every agent-to-agent handoff path (including return paths) to have a corresponding schema file, **so that** I can understand and validate what data flows between any two agents.

**Why P1**: Missing return path schemas mean the reviewer-to-orchestrator, coder-to-orchestrator, and docs-to-orchestrator handoffs are undocumented. Maintainers modifying these boundaries have no contract to reference.

**Independent Test**: Count the total directional handoff paths in the pipeline and verify a schema file exists for each one in `.github/schemas/`.

**Acceptance Scenarios**:
1. **Given** a maintainer lists `.github/schemas/`, **When** they look for reviewer-to-orchestrator.schema.yaml, **Then** it exists and follows handoff/v1 format. (FR-021, FR-027)
2. **Given** the reviewer-to-orchestrator schema exists, **When** a maintainer reads it, **Then** they find required fields: wp_path, verdict, updated_lane. (FR-022)
3. **Given** the docs-agent-to-orchestrator schema exists, **When** a maintainer reads it, **Then** they find required fields: wp_path, docs_completed. (FR-026)

### Edge Cases
- What if a new agent is added that creates a new handoff path? The schema versioning protocol (C9) and developer guide should document the requirement to create a matching schema.

---

### US-09 -- Documented Acceptance Criteria RACI (Priority: P1) MVP

**As a** Pipeline Maintainer, **I want** the Coder/Reviewer responsibility for acceptance criteria checkboxes to be explicitly documented as a maker/checker pattern, **so that** the apparent duplication is understood as intentional dual-touch.

**Why P1**: Without documentation, maintainers may "fix" the perceived duplication by removing one side's checkbox responsibility, breaking the quality assurance loop.

**Independent Test**: Read the Coder agent file and confirm it labels its checkbox role as "Responsible (maker)." Read the Reviewer agent file and confirm it labels its role as "Accountable/Verifier (checker)."

**Acceptance Scenarios**:
1. **Given** a maintainer reads the Coder agent instructions, **When** they look for acceptance criteria handling, **Then** they find the role labeled "Responsible (maker)." (FR-031)
2. **Given** a maintainer reads the developer guide, **When** they look for acceptance criteria ownership, **Then** they find a section explaining the maker/checker pattern. (FR-033)

### Edge Cases
- What if a maintainer disagrees with the maker/checker pattern and wants a single-owner model? The documented design decision provides rationale; changes should go through the spec process.

---

### US-10 -- Schema Version History (Priority: P2)

**As a** Pipeline Maintainer, **I want** each schema file to have a version history section and follow documented versioning rules, **so that** I know when schemas changed and whether a change is breaking.

**Why P2**: Without versioning, a schema change can silently break agents that depend on the old format. Not critical for initial deployment but important for ongoing maintenance.

**Independent Test**: Read any schema file and verify it contains a `version_history` section with at least one entry.

**Acceptance Scenarios**:
1. **Given** a maintainer opens `coder-to-reviewer.schema.yaml`, **When** they scroll to the end, **Then** they find a `version_history` array with the initial version entry. (FR-044, FR-045)
2. **Given** a maintainer removes a required field from a schema, **When** they follow the versioning protocol, **Then** they increment from handoff/v1 to handoff/v2 and add a version_history entry. (FR-046)
3. **Given** a maintainer adds a new optional field, **When** they follow the versioning protocol, **Then** they keep the version as handoff/v1 and add a version_history entry. (FR-047)

### Edge Cases
- What if version_history is missing from a schema? Validation logs a warning but does not halt -- graceful degradation for pre-hardening schemas.

---

### US-11 -- Reusable Schema Validation Patterns (Priority: P2)

**As a** Pipeline Maintainer, **I want** common validation patterns (WP file exists, lane value is valid, path format matches) extracted into a shared base schema, **so that** I do not need to duplicate the same validation rules in every schema file.

**Why P2**: Duplicated validation rules drift over time. When one schema's WP existence check is updated, others remain stale. Not blocking for MVP but improves long-term maintainability.

**Independent Test**: Read `base-handoff.schema.yaml` and verify it defines at least 3 reusable validation patterns. Read at least two individual schemas and verify they reference the base.

**Acceptance Scenarios**:
1. **Given** a maintainer reads `base-handoff.schema.yaml`, **When** they look for validation patterns, **Then** they find: wp_file_exists, lane_value_valid, file_path_format. (FR-050)
2. **Given** the base schema is missing, **When** an individual schema tries to reference it, **Then** the agent falls back to inline validation rules and logs a warning. (FR-049)

### Edge Cases
- What if a specific schema needs a validation rule not in the base? It defines the rule inline alongside its base references.

---

### US-12 -- Mid-Cycle Pattern Propagation (Priority: P3)

**As a** Pipeline Maintainer, **I want** pattern file changes to take effect during the current pipeline run (not just the next one), **so that** the Review Coordinator's newly discovered patterns are applied immediately to subsequent skill dispatches.

**Why P3**: This is an optimization. The current behavior (patterns take effect next invocation) is correct. Mid-cycle propagation only matters for very long pipeline runs where the Review Coordinator finds patterns that a later-dispatched skill should avoid.

**Independent Test**: Start a pipeline run, have the Review Coordinator add a pattern, verify the next skill dispatch within the same run uses the updated patterns.

**Acceptance Scenarios**:
1. **Given** the Review Coordinator adds a pattern to code-patterns.md and increments `patterns_version`, **When** the Coder coordinator dispatches the next skill, **Then** it detects the version change and reloads patterns before dispatch. (FR-053, FR-054)
2. **Given** the patterns file is unreadable on re-check, **When** the coordinator tries to reload, **Then** it uses the last successfully read patterns and logs a warning. (FR-054)

### Edge Cases
- What if patterns_version is missing from the file? The coordinator treats the version as 0 and always reloads (safe default).

---

### US-13 -- Contract-First Validation Against Real Code (Priority: P3)

**As a** Pipeline Maintainer, **I want** a documented pilot test that validates the Coder's contract-first workflow against real source code, **so that** I have confidence the pipeline works beyond self-referential markdown builds.

**Why P3**: The pipeline has only built itself (markdown/YAML). Confidence that it works with real code (TypeScript, Python) requires explicit validation. This is a future investment, not a blocker.

**Independent Test**: Read `.sdd/tests/contract-validation-pilot.md` and verify it contains test scenarios, expected outcomes, and (if executed) actual results.

**Acceptance Scenarios**:
1. **Given** the test document exists, **When** a maintainer reads it, **Then** they find scenarios for contract-first implementation, coverage enforcement, and debug retry loops. (FR-056)
2. **Given** no suitable test codebase is available, **When** the test document is created, **Then** the results section states "Not yet executed" with a reason. (FR-057)

### Edge Cases
- What if the Coder fails on real code due to a language-specific issue? The failure is recorded as a blocking finding with the specific language and error details.

---

**FR-US Coverage Notes** (for traceability skill):
- FR-003: Covered by US-02 (docs_completed field definition). Also referenced in US-01 edge case for `review_cycles`.
- FR-009: Covered by US-06 (enum registry defines named groups).
- FR-014: Covered by US-06 (enum reference comments in agent files). Infrastructure requirement, not user-facing.
- FR-016: Covered by US-07 (canonical format).
- FR-020: Covered by US-07 (conventions in enums.yaml).
- FR-027: Covered by US-08 (handoff/v1 format compliance).
- FR-030: Covered by US-03 (agent file reference comments). Infrastructure requirement, not user-facing.
- FR-039: Not covered directly. Infrastructure detail for configurable thresholds (spec-test-strategy update). Covered transitively via US-04.
- FR-043: Not covered directly. Trivial default case for dependency ordering (no deps = eligible). Covered transitively via US-05.
- FR-044 through FR-048: Covered by US-10 and US-11.
- FR-052 through FR-054: Covered by US-12 and US-13.

---

## 6. User Flows

### 6.1 Review Cycle Tracking and Escalation

**Actor**: Orchestrator Agent
**Precondition**: A WP has been implemented by the Coder and submitted for review (lane=for_review).
**Trigger**: The Review Coordinator completes a review with verdict "Changes Required."

1. **Review Coordinator** sets WP `lane: to_do` and increments `review_cycles` by 1 in WP frontmatter.
2. **Review Coordinator** appends Activity Log entry: `<timestamp> - review-coordinator - lane=to_do - Changes required (FB-XX, FB-YY)`.
3. **Orchestrator** reads updated WP frontmatter.
4. **Orchestrator** reads `review_cycles` from frontmatter.
   - *If `review_cycles` < 3*: Orchestrator invokes the Coder to fix feedback items. Go to step 6.
   - *If `review_cycles` >= 3*: Orchestrator escalates to the user. Go to step 5.
5. **Orchestrator** presents escalation to user via askQuestions with all review feedback from all cycles. **Flow ends** (awaiting user decision).
6. **Coder** addresses feedback, sets `lane: for_review`.
7. **Orchestrator** re-invokes Review Coordinator. Return to step 1.

**Postcondition**: WP is either escalated to the user or proceeds through another review cycle, with `review_cycles` accurately reflecting the count.

---

### 6.2 Documentation Completion Tracking

**Actor**: Orchestrator Agent
**Precondition**: A WP has been reviewed and approved (lane=done).
**Trigger**: Orchestrator determines the next action for a WP with lane=done.

1. **Orchestrator** reads WP frontmatter.
2. **Orchestrator** checks `docs_completed` field.
   - *If `docs_completed` is true*: Skip documentation, advance to next WP or completion check. Go to step 6.
   - *If `docs_completed` is false or absent*: Invoke Docs Agent. Go to step 3.
3. **Orchestrator** invokes Docs Agent with WP path.
4. **Docs Agent** generates documentation (dispatching doc skills).
5. **Docs Agent** sets `docs_completed: true` in WP frontmatter. Returns to Orchestrator.
6. **Orchestrator** re-reads state and proceeds to the next WP or pipeline completion.

**Postcondition**: WP has `docs_completed: true` in frontmatter and documentation has been generated.

---

### 6.3 Adding a New Enum Value

**Actor**: Pipeline Maintainer
**Precondition**: `.github/schemas/enums.yaml` exists with current enum definitions.
**Trigger**: A maintainer needs to add a new valid value to an enum group (e.g., a new lane value).

1. **Maintainer** opens `.github/schemas/enums.yaml`.
2. **Maintainer** adds the new value to the appropriate enum group.
3. **Maintainer** updates all agent files that handle the enum group to recognize the new value.
4. **Maintainer** updates relevant handoff schemas if the new value affects validation rules.
5. **Maintainer** follows the schema versioning protocol:
   - *If the change is additive (new value)*: Keep existing schema version, add version_history entry.
   - *If the change removes or renames a value*: Increment schema version, add version_history entry.
6. **Maintainer** commits all changes together.
7. **System** (Review Coordinator on next review) verifies enum consistency across files.

**Postcondition**: New enum value is defined in the registry and recognized by all consuming agents and schemas.

---

### 6.4 Overriding Coverage Thresholds for a WP

**Actor**: Pipeline Operator
**Precondition**: A spec has been planned into WPs. The operator wants to set non-default coverage thresholds for a specific WP.
**Trigger**: Operator (or the Planner during WP creation) sets coverage fields in WP frontmatter.

1. **Operator** edits WP frontmatter to add `coverage_code: 60` and `coverage_branch: 70`.
2. **Orchestrator** invokes Coder for the WP.
3. **Coder** dispatches code-env-setup skill.
4. **code-env-setup** reads WP frontmatter, finds `coverage_code: 60` and `coverage_branch: 70`.
5. **code-env-setup** configures test tooling with 60%/70% thresholds.
6. **Coder** dispatches code-unit-tests skill.
7. **code-unit-tests** reads WP frontmatter, finds the same override values.
8. **code-unit-tests** enforces 60%/70% thresholds instead of 80%/90% defaults.
   - *If coverage meets thresholds*: Skill reports success.
   - *If coverage below thresholds*: Skill reports failure; Coder dispatches debug skill.

**Postcondition**: Coverage enforcement uses WP-specific thresholds. WP advances through the pipeline with project-appropriate coverage requirements.

---

### 6.5 Dependency-Aware WP Selection

**Actor**: Orchestrator Agent
**Precondition**: Multiple WPs exist with `lane: planned`. Some have `depends_on` fields.
**Trigger**: Orchestrator is selecting the next WP to implement.

1. **Orchestrator** reads all WP files and extracts `lane` and `depends_on` from frontmatter.
2. **Orchestrator** builds a dependency graph from `depends_on` references.
3. **Orchestrator** checks for circular dependencies.
   - *If circular dependency found*: Halt with cycle description. **Flow ends.**
4. **Orchestrator** checks for references to non-existent WPs.
   - *If invalid reference found*: Halt with error. **Flow ends.**
5. **Orchestrator** performs topological sort of WPs with `lane: planned`.
6. **Orchestrator** filters to WPs whose dependencies all have `lane: done`.
   - *If no WPs are eligible*: Report blocked status. **Flow ends.**
7. **Orchestrator** selects the lowest-numbered WP among eligible WPs (tiebreaker).
8. **Orchestrator** invokes Coder for the selected WP.

**Postcondition**: The selected WP has all dependencies met. WPs are executed in a valid topological order.

---

### 6.6 Schema Modification with Versioning

**Actor**: Pipeline Maintainer
**Precondition**: A handoff schema exists that needs modification.
**Trigger**: Agent interface change requires schema update.

1. **Maintainer** identifies the change type:
   - *If adding optional fields*: Additive change. Go to step 2.
   - *If removing/renaming fields or changing types*: Breaking change. Go to step 3.
2. **Maintainer** adds fields to the schema. Keeps `schema: handoff/v1` unchanged. Adds a `version_history` entry with the date and description. Go to step 4.
3. **Maintainer** modifies the schema. Changes `schema: handoff/v1` to `schema: handoff/v2`. Adds a `version_history` entry. Updates all agents that consume this schema to reference the new version.
4. **Maintainer** verifies the base schema reference is present (if applicable).
5. **Maintainer** commits the schema change with agent updates.

**Postcondition**: Schema is updated with a version history entry. If breaking, all consuming agents are updated to the new version.

---

## 7. Data Model

This spec modifies configuration files (markdown and YAML), not application databases. The "entities" below represent the structured data formats defined or modified by this hardening pass. Each entity describes the YAML/markdown schema for a file type that agents read and write.

### 7.1 WP Frontmatter (Extended)

Extends the existing WP file YAML frontmatter with new optional fields for review tracking, documentation tracking, and configurable coverage thresholds.

| Field | Type | Required | Constraints | Default | Description |
|-------|------|----------|-------------|---------|-------------|
| lane | string (enum) | yes | One of: planned, doing, for_review, to_do, done, blocked | planned | Current WP lifecycle state |
| review_status | string (enum) | no | One of: pending, has_feedback, acknowledged, approved | pending | Review state within the review cycle |
| depends_on | array of string | no | Each element matches `WP\d{2}` format | [] | WP identifiers this WP depends on |
| review_cycles | integer | no | >= 0 | 0 | Number of review-rework cycles completed |
| docs_completed | boolean | no | true or false | false | Whether documentation has been generated |
| coverage_code | integer | no | 0-100 | 80 | Minimum code coverage percentage override |
| coverage_branch | integer | no | 0-100 | 90 | Minimum branch coverage percentage override |

**Note**: Fields `lane`, `review_status`, and `depends_on` are existing fields. Fields `review_cycles`, `docs_completed`, `coverage_code`, and `coverage_branch` are new additions from this spec. Existing WP files without the new fields remain valid; agents treat absent fields as their defaults.

#### Relationships

| Related Entity | Cardinality | Description |
|---------------|-------------|-------------|
| Enum Registry | N:1 | `lane` and `review_status` values are defined in enums.yaml |
| Handoff Schema | N:1 | WP state transitions are validated against handoff schemas |

#### Validation Rules

- `review_cycles` SHALL be non-negative. Negative values are treated as 0.
- `coverage_code` and `coverage_branch` SHALL be in range [0, 100]. Out-of-range values cause a halt.
- `depends_on` entries SHALL reference existing WP files. Non-existent references cause a halt.
- If `lane` is not one of the enum values, the reading agent SHALL halt.

#### State Machine: WP lane

**States**: planned, doing, for_review, to_do, done, blocked

| From | To | Guard | Side Effects |
|------|-----|-------|-------------|
| planned | doing | Coder selects WP; dependencies met | Coder appends Activity Log entry |
| doing | for_review | All tasks complete, tests passing | Coder appends Activity Log entry |
| for_review | done | Reviewer verdict: approved | Reviewer appends Activity Log entry |
| for_review | to_do | Reviewer verdict: changes_required | Reviewer increments `review_cycles` by 1; appends Activity Log entry |
| to_do | doing | Coder acknowledges feedback | Coder sets review_status=acknowledged; appends Activity Log entry |
| any | blocked | Unresolvable blocker identified | Agent appends Activity Log entry |
| blocked | planned | Blocker resolved | Manually or by Orchestrator; appends Activity Log entry |

**Invalid transitions**: Any transition not listed above SHALL be rejected. Agents SHALL NOT set lane to a value that is not reachable from the current value.

---

### 7.2 Enum Registry

The central enum registry file at `.github/schemas/enums.yaml` defining all enum values and conventions used across the pipeline.

| Field | Type | Required | Constraints | Default | Description |
|-------|------|----------|-------------|---------|-------------|
| lane | array of string | yes | Exactly: planned, doing, for_review, to_do, done, blocked | - | Valid WP lane values |
| spec_status | array of string | yes | Exactly: Draft, Validated, Approved | - | Valid spec status values |
| pipeline_stage | array of string | yes | Exactly: idle, ideation, specification, planning, implementation, review, documentation, complete | - | Valid Orchestrator pipeline stages |
| review_status | array of string | yes | Exactly: pending, has_feedback, acknowledged, approved | - | Valid review status values |
| conventions.activity_log_format | string | yes | Template string with placeholders | - | Canonical Activity Log entry format |

#### Relationships

| Related Entity | Cardinality | Description |
|---------------|-------------|-------------|
| WP Frontmatter | 1:N | All WP files reference lane and review_status enums |
| Handoff Schema | 1:N | Schemas reference enum values for validation |
| Orchestrator State | 1:1 | state.md references pipeline_stage enum |

#### Validation Rules

- Each enum group SHALL contain at least one value.
- Enum values SHALL be unique within their group.
- The conventions section SHALL contain the `activity_log_format` template.

---

### 7.3 Handoff Schema

The structure of each handoff schema file in `.github/schemas/`. Applies to both existing forward schemas and new return schemas.

| Field | Type | Required | Constraints | Default | Description |
|-------|------|----------|-------------|---------|-------------|
| schema | string | yes | Format: `handoff/v\d+` | handoff/v1 | Schema version identifier |
| source_agent | string | yes | Valid agent name | - | Producing agent |
| target_agent | string | yes | Valid agent name | - | Consuming agent |
| description | string | yes | Non-empty | - | Human-readable description |
| base_schema | string | no | Valid file path to base-handoff.schema.yaml | - | Reference to shared base schema |
| required_artifacts | array of object | yes | At least one entry | - | Artifacts the source must produce |
| required_state | array of object | yes | At least one condition | - | Conditions for valid handoff |
| context_fields | array of object | yes | At least one field | - | Data passed in the handoff prompt |
| validation_rules | array of object | yes | At least one rule | - | Checks target agent runs |
| version_history | array of object | no | Each entry has version, date, description | - | Record of schema changes |
| placeholder_patterns | object | no | Keys are placeholder names, values are regex | - | Regex patterns for artifact path placeholders |

#### Relationships

| Related Entity | Cardinality | Description |
|---------------|-------------|-------------|
| Base Handoff Schema | N:1 | Individual schemas reference the base for common patterns |
| Enum Registry | N:1 | Validation rules reference enum values |

#### Validation Rules

- `schema` field SHALL match the regex `handoff/v\d+`.
- `source_agent` and `target_agent` SHALL be valid pipeline agent names.
- If `base_schema` is specified, the referenced file SHALL exist.
- `version_history` entries SHALL be ordered chronologically (newest last).

---

### 7.4 Base Handoff Schema

The shared base schema at `.github/schemas/base-handoff.schema.yaml` defining reusable validation patterns.

| Field | Type | Required | Constraints | Default | Description |
|-------|------|----------|-------------|---------|-------------|
| schema | string | yes | `base/v1` | base/v1 | Base schema version |
| description | string | yes | Non-empty | - | Purpose of the base schema |
| validation_patterns | object | yes | At least one named pattern | - | Reusable validation pattern definitions |
| validation_patterns.wp_file_exists | object | yes | Has `check` and `error` fields | - | Validates WP file exists with valid YAML |
| validation_patterns.lane_value_valid | object | yes | Has `check`, `enum_ref`, and `error` fields | - | Validates lane is a valid enum value |
| validation_patterns.file_path_format | object | yes | Has `check`, `pattern`, and `error` fields | - | Validates file paths match expected format |

#### Relationships

| Related Entity | Cardinality | Description |
|---------------|-------------|-------------|
| Handoff Schema | 1:N | Referenced by individual schemas via `base_schema` field |
| Enum Registry | 1:1 | `lane_value_valid` references the lane enum |

#### Validation Rules

- Each validation pattern SHALL have a `check` field describing what it validates.
- Each validation pattern SHALL have an `error` field with the error message template.
- The `lane_value_valid` pattern SHALL reference enums.yaml for valid values.

---

### 7.5 Pattern File Frontmatter (Extended)

Extends the existing patterns file format with a version counter for propagation tracking.

| Field | Type | Required | Constraints | Default | Description |
|-------|------|----------|-------------|---------|-------------|
| patterns_version | integer | no | >= 1 | 1 | Version counter incremented on each pattern change |

#### Relationships

| Related Entity | Cardinality | Description |
|---------------|-------------|-------------|
| Coordinator Agents | 1:N | Coordinators check patterns_version before skill dispatch |

#### Validation Rules

- `patterns_version` SHALL be a positive integer.
- If absent, coordinators SHALL treat it as 0 (always reload).

---

### 7.6 Activity Log Entry

The canonical format for log entries appended to WP files. This is a text format, not a YAML structure.

| Field | Type | Required | Constraints | Default | Description |
|-------|------|----------|-------------|---------|-------------|
| timestamp | string | yes | ISO 8601 format (e.g., 2026-04-06T14:30:00Z) | current time | When the action occurred |
| agent_name | string | yes | One of: coder, review-coordinator, docs-agent, orchestrator | - | Which agent wrote the entry |
| action | string | yes | Non-empty, typically lane=<value> or descriptive action | - | What action was taken |
| details | string | yes | Non-empty, max 200 characters | - | Human-readable description |

#### Relationships

| Related Entity | Cardinality | Description |
|---------------|-------------|-------------|
| WP Frontmatter | N:1 | Log entries are appended to the WP file's Activity Log section |

#### Validation Rules

- Fields SHALL be separated by ` - ` (space-hyphen-space).
- The resulting entry SHALL match: `<ISO-8601> - <agent-name> - <action> - <details>`.
- `agent_name` SHALL use lowercase with hyphens (not spaces or camelCase).

---

### 7.7 Return Handoff Context

The context fields specific to return handoff schemas (reviewer-to-orchestrator, coder-to-orchestrator, docs-to-orchestrator).

**Reviewer-to-Orchestrator context fields**:

| Field | Type | Required | Constraints | Default | Description |
|-------|------|----------|-------------|---------|-------------|
| wp_path | string | yes | Valid file path to WP file | - | Path to the reviewed WP |
| verdict | string (enum) | yes | One of: approved, changes_required | - | Review outcome |
| updated_lane | string (enum) | yes | One of: done, to_do | - | Lane value set by reviewer |

**Coder-to-Orchestrator context fields**:

| Field | Type | Required | Constraints | Default | Description |
|-------|------|----------|-------------|---------|-------------|
| wp_path | string | yes | Valid file path to WP file | - | Path to the implemented WP |
| lane_confirmation | string | yes | Literal value: `for_review` | - | Confirms lane was set to for_review |

**Docs-Agent-to-Orchestrator context fields**:

| Field | Type | Required | Constraints | Default | Description |
|-------|------|----------|-------------|---------|-------------|
| wp_path | string | yes | Valid file path to WP file | - | Path to the documented WP |
| docs_completed | boolean | yes | true | - | Confirms documentation was generated |

#### Relationships

| Related Entity | Cardinality | Description |
|---------------|-------------|-------------|
| Handoff Schema | 1:1 | Each return context maps to one schema file |
| WP Frontmatter | N:1 | Return handoffs reference WP files |

#### Validation Rules

- `verdict` SHALL be one of: approved, changes_required.
- `updated_lane` SHALL match `verdict`: approved->done, changes_required->to_do.
- `lane_confirmation` SHALL be literally `for_review`.
- `docs_completed` SHALL be true (false is not a valid completion signal).

---

## 8. Agent Interfaces

This system has no HTTP APIs. Agents interact via file reads/writes (YAML frontmatter, schema files, pattern files) and structured handoff prompts. This section defines the public interfaces that agents use to read and write pipeline state.

### 8.1 WP Frontmatter Read/Write Interface

**Purpose**: Read and write structured state fields in WP file YAML frontmatter.

**Consumers**: Orchestrator (read), Coder (read/write), Review Coordinator (read/write), Docs Agent (read/write)

**Read Operations**:

| Operation | Input | Output | Source FR |
|-----------|-------|--------|-----------|
| Read review_cycles | WP file path (string) | integer (default 0 if absent) | FR-005 |
| Read docs_completed | WP file path (string) | boolean (default false if absent) | FR-006 |
| Read coverage_code | WP file path (string) | integer (default 80 if absent) | FR-036 |
| Read coverage_branch | WP file path (string) | integer (default 90 if absent) | FR-036 |
| Read lane | WP file path (string) | string enum (see Section 7.2) | Existing |
| Read depends_on | WP file path (string) | array of string (default [] if absent) | FR-040 |

**Write Operations**:

| Operation | Input | Output | Source FR |
|-----------|-------|--------|-----------|
| Increment review_cycles | WP file path (string) | review_cycles = previous + 1 | FR-002 |
| Set docs_completed | WP file path (string), value: true | docs_completed = true | FR-004 |
| Set lane | WP file path (string), new lane value (string enum) | lane = new value | Existing |

**Errors**:

| Code | Error Key | Condition | Response |
|------|-----------|-----------|----------|
| E-001 | WP_FILE_NOT_FOUND | WP file does not exist at path | Halt with "WP file not found at {path}" |
| E-002 | INVALID_FRONTMATTER | YAML frontmatter cannot be parsed | Halt with "Invalid YAML frontmatter in {path}" |
| E-003 | INVALID_FIELD_TYPE | Field value has unexpected type | Log warning, use default value |
| E-004 | COVERAGE_OUT_OF_RANGE | coverage_code or coverage_branch not in [0, 100] | Halt with "Invalid coverage value '{value}'. Must be integer 0-100." |
| E-005 | INVALID_LANE_VALUE | lane value not in enum registry | Halt with "Invalid lane value '{value}'. Valid values: {enum_values}" |
| E-006 | FIELD_ABSENT | New field not present in pre-hardening WP file | Use default value (no error) |

---

### 8.2 Enum Registry Read Interface

**Purpose**: Read enum values and conventions from the central registry.

**Consumers**: All agents and schema validators

**Read Operations**:

| Operation | Input | Output | Source FR |
|-----------|-------|--------|-----------|
| Read enum group | Group name (string) | array of string values | FR-009 |
| Read activity log format | None | format template string | FR-020 |
| Validate enum value | Group name (string), value (string) | boolean (valid/invalid) | FR-010-013 |

**Errors**:

| Code | Error Key | Condition | Response |
|------|-----------|-----------|----------|
| E-010 | ENUM_REGISTRY_NOT_FOUND | enums.yaml missing | Halt with "Enum registry not found at .github/schemas/enums.yaml" |
| E-011 | ENUM_GROUP_NOT_FOUND | Requested group not in file | Halt with "Enum group '{name}' not found in enums.yaml" |
| E-012 | INVALID_ENUM_VALUE | Value not in enum group | Halt with "Invalid {group} value '{value}'. Valid values: {values}" |
| E-013 | ENUM_PARSE_ERROR | enums.yaml is malformed YAML | Halt with YAML parse error details |

---

### 8.3 Handoff Schema Validation Interface

**Purpose**: Validate agent-to-agent handoff data against schema files.

**Consumers**: All coordinator agents at startup

**Read Operations**:

| Operation | Input | Output | Source FR |
|-----------|-------|--------|-----------|
| Load schema | Schema file path (string) | Parsed schema object | FR-027 |
| Validate required artifacts | Schema object, actual file paths | boolean (pass/fail) + error list | FR-027 |
| Validate required state | Schema object, current state | boolean (pass/fail) + error list | FR-027 |
| Validate context fields | Schema object, handoff prompt | boolean (pass/fail) + missing field list | FR-022, FR-024, FR-026 |
| Load base schema | base_schema path from schema | Base validation patterns | FR-051 |

**Errors**:

| Code | Error Key | Condition | Response |
|------|-----------|-----------|----------|
| E-020 | SCHEMA_NOT_FOUND | Schema file missing | Halt with "Schema file not found at {path}" |
| E-021 | MISSING_CONTEXT_FIELD | Required context field absent from handoff | Halt with "Missing required field '{name}' in {schema_name} handoff" |
| E-022 | INVALID_FIELD_VALUE | Field value does not match expected | Halt with expected vs actual values |
| E-023 | ARTIFACT_NOT_FOUND | Required artifact file missing | Halt with "Required artifact not found at {path}" |
| E-024 | BASE_SCHEMA_NOT_FOUND | base-handoff.schema.yaml missing | Warning; fall back to inline rules |
| E-025 | SCHEMA_PARSE_ERROR | Schema file is malformed YAML | Halt with YAML parse error details |

---

### 8.4 Pattern File Read Interface

**Purpose**: Read and version-check domain patterns files for mid-cycle propagation.

**Consumers**: Coordinator agents (Spec Architect, Planner, Coder, Docs Agent)

**Read Operations**:

| Operation | Input | Output | Source FR |
|-----------|-------|--------|-----------|
| Read patterns content | Pattern file path (string) | Patterns markdown text | FR-054 |
| Read patterns_version | Pattern file path (string) | integer (default 0 if absent) | FR-052 |
| Check version changed | Pattern file path, last-known version (integer) | boolean | FR-054 |

**Write Operations** (Review Coordinator only):

| Operation | Input | Output | Source FR |
|-----------|-------|--------|-----------|
| Add pattern | Pattern file path, pattern text | Updated file with new pattern | FR-053 |
| Increment patterns_version | Pattern file path | patterns_version = previous + 1 | FR-053 |

**Errors**:

| Code | Error Key | Condition | Response |
|------|-----------|-----------|----------|
| E-030 | PATTERNS_FILE_NOT_FOUND | Patterns file missing | Set patterns to "No active patterns"; log warning |
| E-031 | PATTERNS_UNREADABLE | File exists but cannot be read | Use last cached patterns; log warning |
| E-032 | PATTERNS_VERSION_INVALID | patterns_version is not an integer | Treat as 0 (always reload) |

---

### 8.5 Activity Log Write Interface

**Purpose**: Append standardized log entries to WP files.

**Consumers**: Coder, Review Coordinator, Docs Agent

**Write Operation**:

| Operation | Input | Output | Source FR |
|-----------|-------|--------|-----------|
| Append log entry | WP file path, agent_name (string), action (string), details (string) | Log entry appended to Activity Log section | FR-016 |

**Format**: `<ISO-8601-timestamp> - <agent_name> - <action> - <details>`

**Errors**:

| Code | Error Key | Condition | Response |
|------|-----------|-----------|----------|
| E-040 | ACTIVITY_LOG_SECTION_MISSING | WP file has no Activity Log section | Create the section heading, then append entry |
| E-041 | WP_FILE_WRITE_FAILURE | Cannot write to WP file | Halt (critical-path agents) or log warning (advisory agents) |

---

### 8.6 Dependency Graph Interface

**Purpose**: Build and query the WP dependency graph for topological ordering.

**Consumer**: Orchestrator

**Operations**:

| Operation | Input | Output | Source FR |
|-----------|-------|--------|-----------|
| Build graph | Array of WP file paths with frontmatter | Directed acyclic graph | FR-040 |
| Topological sort | Dependency graph | Ordered list of WP identifiers | FR-040 |
| Detect cycles | Dependency graph | Cycle description or empty | FR-042 |
| Get eligible WPs | Sorted WPs, current lane values | List of WPs with all deps met and lane=planned | FR-041 |

**Errors**:

| Code | Error Key | Condition | Response |
|------|-----------|-----------|----------|
| E-050 | CIRCULAR_DEPENDENCY | Cycle detected in dependency graph | Halt with "Circular dependency detected: {cycle}" |
| E-051 | MISSING_DEPENDENCY | depends_on references non-existent WP | Halt with "{wp} depends on {dep} which does not exist" |
| E-052 | ALL_WPS_BLOCKED | No WPs eligible (all have unmet deps) | Report "No WPs ready. Blocked WPs: {list}" |

---

## 9. Architecture

### 9.1 System Design

The SDD pipeline is a sequential, agent-based instruction processing system. There is no compiled runtime, no server, and no database. "Components" are markdown instruction files (.agent.md) and skill files (SKILL.md) interpreted by LLM agents within VS Code Copilot Chat. State is stored in YAML-frontmatted markdown files in the `.sdd/` directory. Agent-to-agent communication is mediated by handoff schemas in `.github/schemas/`.

This hardening pass preserves the existing architecture. It adds new configuration/registry files and modifies existing instruction files but does not change the component topology.

**Components**:

| Component | Responsibility | File Type |
|-----------|---------------|-----------|
| Agents (8) | Coordinate skill dispatch, manage state transitions | `.github/agents/*.agent.md` |
| Skills (37) | Execute domain-specific tasks as subagents | `.github/skills/*/SKILL.md` |
| Handoff Schemas (8 + 3 new) | Define agent-to-agent data contracts | `.github/schemas/*.schema.yaml` |
| Skill Contracts (4) | Define skill input/output standards | `.github/skills/*-SKILL-CONTRACT.md` |
| Enum Registry (new) | Central source of truth for all enum values | `.github/schemas/enums.yaml` |
| Base Schema (new) | Shared validation patterns for schemas | `.github/schemas/base-handoff.schema.yaml` |
| Pipeline State | Persistent orchestrator state | `.sdd/state.md` |
| WP Files | Work package state and activity logs | `.sdd/plans/WP*.md` |
| Pattern Files | Recurring issues per domain | `.sdd/reviews/*-patterns.md` |
| Documentation | Architecture, API reference, guides | `.sdd/docs/*.md` |

**Interaction Flow**:

```
Orchestrator -> reads state.md, WP frontmatter, enums.yaml
           |-> delegates to Agent (via handoff schema)
           |-> Agent reads skill contract, patterns file (checks patterns_version)
           |-> Agent dispatches Skill (passes patterns, WP context)
           |-> Skill reads/writes WP frontmatter, appends Activity Log
           |-> Agent signals completion (via return handoff schema)
           |-> Orchestrator reads updated state, decides next action
```

---

### 9.2 Technology Stack

| Layer | Technology | Version | Rationale |
|-------|-----------|---------|-----------|
| Instruction format | Markdown with YAML frontmatter | CommonMark + YAML 1.2 | Native VS Code Copilot format |
| Schema format | YAML | 1.2 | Human-readable, supports anchors/aliases for base schema references |
| Agent runtime | VS Code Copilot Chat (Claude Opus 4.6) | Current | LLM-based instruction interpreter |
| State storage | Flat files in `.sdd/` | N/A | No database; all state is in version-controlled files |
| Version control | Git | 2.x | All deliverables are text files tracked in Git |

No package isolation is required because there are no software dependencies. All deliverables are markdown and YAML files.

---

### 9.3 Directory & Module Structure

Files modified or created by this spec are marked with `[modified]` or `[new]`:

```
.github/
  agents/
    orchestrator.agent.md       [modified] -- frontmatter reads, topological sort, error policy ref
    coder.agent.md              [modified] -- log format, RACI label, coverage reads, error policy ref
    review-coordinator.agent.md [modified] -- review_cycles increment, log format, RACI label, error policy ref
    docs-agent.agent.md         [modified] -- docs_completed write, log format, error policy ref
    planner.agent.md            [modified] -- remove "Final" reference, error policy ref
    spec-architect.agent.md     [modified] -- patterns_version check, error policy ref
    ideation.agent.md           [modified] -- error policy ref
    brainstorming.agent.md      [modified] -- error policy ref
  schemas/
    enums.yaml                  [new] -- central enum registry (C2)
    base-handoff.schema.yaml    [new] -- shared validation patterns (C10)
    reviewer-to-orchestrator.schema.yaml    [new] -- return handoff (C4)
    coder-complete-to-orchestrator.schema.yaml  [new] -- return handoff (C4)
    docs-agent-to-orchestrator.schema.yaml  [new] -- return handoff (C4)
    coder-to-reviewer.schema.yaml           [modified] -- version_history, base ref
    ideation-to-spec.schema.yaml            [modified] -- version_history, base ref
    orchestrator-handoff.schema.yaml        [modified] -- version_history, base ref
    planner-to-coder.schema.yaml            [modified] -- version_history, base ref
    planner-to-spec.schema.yaml             [modified] -- version_history, base ref
    reviewer-to-coder.schema.yaml           [modified] -- version_history, base ref
    reviewer-to-spec.schema.yaml            [modified] -- version_history, base ref
    spec-to-planner.schema.yaml             [modified] -- version_history, base ref
  skills/
    code-unit-tests/SKILL.md    [modified] -- configurable coverage thresholds (C7)
    code-env-setup/SKILL.md     [modified] -- configurable coverage thresholds (C7)
    spec-test-strategy/SKILL.md [modified] -- reference configurable thresholds
.sdd/
  docs/
    architecture.md             [modified] -- error-handling policy design decision (C5)
    developer-guide.md          [modified] -- RACI, schema versioning protocol, enum docs
  reviews/
    spec-patterns.md            [modified] -- add patterns_version frontmatter (C11)
    plan-patterns.md            [modified] -- add patterns_version frontmatter
    code-patterns.md            [modified] -- add patterns_version frontmatter
    doc-patterns.md             [modified] -- add patterns_version frontmatter
  tests/
    contract-validation-pilot.md [new] -- contract validation test document (C12)
```

---

### 9.4 Key Design Decisions

**Decision 1: Frontmatter fields over Activity Log parsing for state queries**
- **Decision**: Use structured YAML frontmatter fields (`review_cycles`, `docs_completed`) instead of parsing Activity Log text entries.
- **Rationale**: Frontmatter fields are machine-readable, type-checked at read time, and do not depend on text format consistency. Activity Logs remain for human-readable audit trails.
- **Alternatives considered**: (a) Structured JSON block in Activity Log -- rejected because it adds complexity to a human-readable section. (b) Separate state tracking file per WP -- rejected because it duplicates the WP file and increases file count.
- **Consequences**: Agents must write both frontmatter field updates AND Activity Log entries (dual-write). Frontmatter is the source of truth; logs are supplementary.
- **Source**: Brief Concern 2-3; Terraform state management analogy.

**Decision 2: Standalone YAML file for enum registry (not embedded in documentation)**
- **Decision**: Create `.github/schemas/enums.yaml` as a standalone file, not a section within architecture docs or a conventions document.
- **Rationale**: Agent instructions can reference a specific file path for lookups. Embedding in documentation would require agents to parse markdown to extract YAML, which is fragile.
- **Alternatives considered**: (a) Embed in `architecture.md` -- rejected for parsing fragility. (b) Embed in each schema file -- rejected as duplication.
- **Consequences**: One additional file to maintain. Adding an enum group means editing this file, not a documentation page.
- **Source**: Brief open question 1.

**Decision 3: Return handoff schemas follow existing handoff/v1 pattern (not lightweight signals)**
- **Decision**: Return schemas use the same structure as forward schemas (schema, source_agent, target_agent, required_artifacts, required_state, context_fields, validation_rules).
- **Rationale**: Consistency with existing schemas reduces cognitive load. The handoff/v1 format is flexible enough for lightweight signals (just fewer required_artifacts).
- **Alternatives considered**: A stripped-down "signal" format -- rejected because maintaining two schema formats doubles the learning curve.
- **Consequences**: Return schemas are slightly more verbose than needed for simple signals, but the consistency benefit outweighs the verbosity cost.
- **Source**: Brief open question 2.

**Decision 4: Coverage threshold overrides in WP frontmatter (not spec-level or project-level)**
- **Decision**: Override `coverage_code` and `coverage_branch` per-WP via frontmatter fields.
- **Rationale**: Thresholds are consumed by skills dispatched per-WP. WP-level overrides are the most granular and closest to the consumption point. Per-spec overrides would require all WPs in a spec to share thresholds, which is too coarse for mixed-priority WPs.
- **Alternatives considered**: (a) Spec frontmatter -- rejected as too coarse. (b) Project-level config file -- rejected as too global. (c) Both spec and WP with WP overriding spec -- rejected as over-engineered for the current need.
- **Consequences**: Each WP can independently set thresholds. If a team wants all WPs to share a threshold, they must set it in each WP (or accept the defaults).
- **Source**: User decision during gap analysis.

**Decision 5: Poll-based pattern propagation (not push-based)**
- **Decision**: Coordinators check `patterns_version` before each skill dispatch and reload if changed, rather than using a push notification or file-watching mechanism.
- **Rationale**: The pipeline has no event system or file-watching infrastructure. Polling at dispatch time is minimal overhead (one file read per dispatch) and guaranteed reliable.
- **Alternatives considered**: (a) File system watcher -- rejected because VS Code extensions required, runtime infra not available. (b) Shared memory/channel -- rejected because agents run in isolated LLM sessions.
- **Consequences**: Pattern changes take effect on the next skill dispatch within the same coordinator session. There is no sub-dispatch-cycle propagation.
- **Source**: Brief C11.

---

### 9.5 External Integrations

No external integrations are required. The SDD pipeline operates entirely within the local filesystem and VS Code Copilot Chat. All file operations are local reads and writes.

---

## 11. Test Requirements

This spec produces markdown and YAML configuration files, not executable software. "Testing" means verifying file contents, cross-file consistency, and pipeline behavior through controlled runs. Traditional unit test coverage metrics do not apply to markdown/YAML files. Instead, each deliverable is validated through content checks, schema compliance, and pipeline integration runs.

**Coverage thresholds**: Configurable per-WP via `coverage_code` and `coverage_branch` frontmatter fields, with defaults of 80% code coverage and 90% branch coverage. These thresholds apply to future WPs that implement executable code using the hardened pipeline, not to this spec's deliverables.

### 11.1 Content Validation Tests

Since deliverables are markdown/YAML files, "unit tests" are content validation checks verifiable by text search and YAML parsing:

| File | Validation | Method |
|------|-----------|--------|
| `.github/schemas/enums.yaml` | Contains all 4 enum groups with exact values | YAML parse + value comparison |
| `.github/schemas/enums.yaml` | `conventions.activity_log_format` is present | YAML parse |
| `.github/schemas/base-handoff.schema.yaml` | Contains 3 named validation patterns | YAML parse |
| `reviewer-to-orchestrator.schema.yaml` | Has schema, source_agent, target_agent, context_fields with wp_path, verdict, updated_lane | YAML parse |
| `coder-complete-to-orchestrator.schema.yaml` | Has context_fields with wp_path, lane_confirmation=for_review | YAML parse |
| `docs-agent-to-orchestrator.schema.yaml` | Has context_fields with wp_path, docs_completed | YAML parse |
| All 11 handoff schemas | Each has a `version_history` section | grep search |
| All 8 agent files | Each has an error policy reference comment | grep search |
| All agent files referencing enums | Each has an enums.yaml reference comment | grep search |
| `orchestrator.agent.md` | No Activity Log scanning for review_cycles or docs_completed | grep search (absence check) |
| `orchestrator.agent.md` | Topological sort algorithm described in WP Selection section | Content review |
| `planner.agent.md` | No "Final" spec status reference | grep search (absence check) |
| `coder.agent.md` | "Responsible (maker)" label for acceptance criteria | grep search |
| `review-coordinator.agent.md` | "Accountable/Verifier (checker)" label | grep search |
| `review-coordinator.agent.md` | Increments `review_cycles` on lane=to_do | Content review |
| `docs-agent.agent.md` | Sets `docs_completed: true` on completion | Content review |
| All 4 pattern files | `patterns_version` in YAML frontmatter | YAML frontmatter parse |
| `code-unit-tests/SKILL.md` | Reads coverage_code/coverage_branch from frontmatter with fallback | Content review |
| `code-env-setup/SKILL.md` | Reads coverage_code/coverage_branch from frontmatter with fallback | Content review |
| `.sdd/docs/architecture.md` | Contains "Design Decision: Error-Handling Policy" section | grep search |
| `.sdd/docs/developer-guide.md` | Contains RACI/maker-checker section | grep search |
| `.sdd/docs/developer-guide.md` | Contains schema versioning protocol section | grep search |

### 11.2 BDD / Acceptance Tests

Gherkin scenarios mapped 1:1 to acceptance scenarios from Section 5.

```gherkin
Feature: Review Cycle Tracking (C1)

  # Source: US-01 Scenario 1
  Scenario: Orchestrator escalates after 3 review cycles via frontmatter
    Given a WP with review_cycles: 2 in frontmatter
    And the Review Coordinator returns verdict "Changes Required"
    When the Review Coordinator updates the WP
    Then review_cycles is incremented to 3
    And the Orchestrator escalates to the user instead of re-invoking the Coder

  # Source: US-01 Scenario 2
  Scenario: Missing review_cycles field treated as 0
    Given a WP file created before the hardening pass with no review_cycles field
    When the Orchestrator evaluates it for escalation
    Then it treats the missing field as 0
    And it does not escalate

  # Source: US-01 Scenario 3
  Scenario: Non-integer review_cycles treated as 0
    Given a WP with review_cycles: "invalid" in frontmatter
    When an agent reads the WP
    Then it treats review_cycles as 0
    And it logs a warning

  # Source: US-01 Edge Case 1
  Scenario: Negative review_cycles treated as 0
    Given a WP with review_cycles: -1 in frontmatter
    When an agent reads the WP
    Then it treats review_cycles as 0
    And it logs a warning

Feature: Documentation Completion Tracking (C1)

  # Source: US-02 Scenario 1
  Scenario: Orchestrator invokes Docs Agent for undocumented WP
    Given a WP with lane: done and docs_completed: false
    When the Orchestrator evaluates next action
    Then it invokes the Docs Agent for that WP

  # Source: US-02 Scenario 2
  Scenario: Missing docs_completed treated as false
    Given a WP with lane: done and no docs_completed field
    When the Orchestrator evaluates next action
    Then it treats the WP as not yet documented
    And it invokes the Docs Agent

  # Source: US-02 Scenario 3
  Scenario: Docs Agent sets docs_completed on completion
    Given the Docs Agent has been invoked for a WP
    When it finishes documentation generation
    Then it sets docs_completed: true in WP frontmatter

  # Source: US-02 Edge Case 1
  Scenario: String "true" treated as false for docs_completed
    Given a WP with docs_completed: "true" (string, not boolean)
    When an agent reads the WP
    Then it treats docs_completed as false
    And it logs a warning

Feature: Error Policy Documentation (C5)

  # Source: US-03 Scenario 1
  Scenario: Architecture doc contains error-handling policy
    Given a user reads .sdd/docs/architecture.md
    When they look for error-handling guidance
    Then they find a "Design Decision: Error-Handling Policy" subsection
    And every agent is categorized as critical-path or advisory

  # Source: US-03 Scenario 2
  Scenario: Agent files reference error policy
    Given a user reads any agent file in .github/agents/
    When they look for error policy
    Then they find a reference comment pointing to the architecture doc

Feature: Configurable Coverage Thresholds (C7)

  # Source: US-04 Scenario 1
  Scenario: WP-specific coverage thresholds are enforced
    Given a WP with coverage_code: 60 and coverage_branch: 70
    When the code-unit-tests skill runs
    Then it enforces 60% code and 70% branch coverage thresholds

  # Source: US-04 Scenario 2
  Scenario: Default thresholds used when no overrides specified
    Given a WP with no coverage fields in frontmatter
    When the code-unit-tests skill runs
    Then it enforces 80% code and 90% branch coverage

  # Source: US-04 Scenario 3
  Scenario: Out-of-range coverage value causes halt
    Given a WP with coverage_code: 150
    When the code-unit-tests skill reads it
    Then it halts with "Invalid coverage_code value '150'. Must be an integer 0-100."

  # Source: US-04 Edge Case 1
  Scenario: Only one coverage field specified uses default for the other
    Given a WP with coverage_code: 60 but no coverage_branch
    When the code-unit-tests skill runs
    Then it enforces 60% code coverage
    And it enforces 90% branch coverage (default)

Feature: Dependency-Aware WP Selection (C8)

  # Source: US-05 Scenario 1
  Scenario: Lower-numbered WP skipped when it has unmet dependencies
    Given WP01 depends on WP02 and both have lane: planned
    When the Orchestrator selects the next WP
    Then it selects WP02 first

  # Source: US-05 Scenario 2
  Scenario: Circular dependency detected and reported
    Given WP01 depends on WP02 and WP02 depends on WP01
    When the Orchestrator builds the dependency graph
    Then it halts with "Circular dependency detected: WP01 -> WP02 -> WP01"

  # Source: US-05 Scenario 3
  Scenario: Non-existent dependency causes halt
    Given WP01 depends on WP99 and WP99 does not exist
    When the Orchestrator builds the dependency graph
    Then it halts with "WP01 depends on WP99 which does not exist"

  # Source: US-05 Edge Case 1
  Scenario: All WPs with no dependencies use lowest-number tiebreaker
    Given WP03, WP01, WP02 all have no dependencies and lane: planned
    When the Orchestrator selects the next WP
    Then it selects WP01 (lowest number)

Feature: Central Enum Registry (C2)

  # Source: US-06 Scenario 1
  Scenario: Enum registry contains all lane values
    Given enums.yaml exists at .github/schemas/
    When a maintainer reads the lane enum group
    Then it contains exactly: planned, doing, for_review, to_do, done, blocked

  # Source: US-06 Scenario 2
  Scenario: No "Final" spec status in Planner
    Given a maintainer searches the Planner agent file
    When they check spec_status references
    Then no "Final" references exist

  # Source: US-06 Scenario 3
  Scenario: Missing enums.yaml causes agent halt
    Given enums.yaml is missing from .github/schemas/
    When an agent starts and tries to reference it
    Then it halts with "Enum registry not found at .github/schemas/enums.yaml"

Feature: Activity Log Format (C3)

  # Source: US-07 Scenario 1
  Scenario: Coder uses canonical Activity Log format
    Given the Coder appends an Activity Log entry
    When the entry is written to a WP file
    Then it matches the format: <ISO-8601> - coder - <action> - <details>

  # Source: US-07 Scenario 2
  Scenario: Review Coordinator uses canonical Activity Log format
    Given the Review Coordinator appends an Activity Log entry
    When the entry is written
    Then it follows the same canonical format

  # Source: US-07 Scenario 3
  Scenario: Canonical format documented in conventions
    Given the canonical format is documented in enums.yaml conventions section
    When a maintainer looks up the format
    Then they find a single authoritative definition

Feature: Complete Handoff Schema Coverage (C4)

  # Source: US-08 Scenario 1
  Scenario: Return schema exists for reviewer-to-orchestrator
    Given a maintainer lists .github/schemas/
    When they look for reviewer-to-orchestrator.schema.yaml
    Then it exists and follows handoff/v1 format

  # Source: US-08 Scenario 2
  Scenario: Reviewer-to-orchestrator schema has required fields
    Given the reviewer-to-orchestrator schema exists
    When a maintainer reads it
    Then they find required context_fields: wp_path, verdict, updated_lane

  # Source: US-08 Scenario 3
  Scenario: Docs-to-orchestrator schema has required fields
    Given the docs-agent-to-orchestrator schema exists
    When a maintainer reads it
    Then they find required context_fields: wp_path, docs_completed

Feature: Acceptance Criteria RACI (C6)

  # Source: US-09 Scenario 1
  Scenario: Coder labeled as Responsible for acceptance criteria
    Given a maintainer reads the Coder agent instructions
    When they look for acceptance criteria handling
    Then they find the role labeled "Responsible (maker)"

  # Source: US-09 Scenario 2
  Scenario: Developer guide documents maker/checker pattern
    Given a maintainer reads the developer guide
    When they look for acceptance criteria ownership
    Then they find a section explaining the maker/checker pattern

Feature: Schema Version History (C9)

  # Source: US-10 Scenario 1
  Scenario: Schema files contain version_history
    Given a maintainer opens coder-to-reviewer.schema.yaml
    When they scroll to the end
    Then they find a version_history array with at least one entry

  # Source: US-10 Scenario 2
  Scenario: Breaking change increments version
    Given a maintainer removes a required field from a schema
    When they follow the versioning protocol
    Then they increment from handoff/v1 to handoff/v2

  # Source: US-10 Scenario 3
  Scenario: Additive change keeps version
    Given a maintainer adds a new optional field
    When they follow the versioning protocol
    Then they keep the version as handoff/v1

Feature: Shared Base Schema (C10)

  # Source: US-11 Scenario 1
  Scenario: Base schema defines reusable patterns
    Given a maintainer reads base-handoff.schema.yaml
    When they look for validation patterns
    Then they find: wp_file_exists, lane_value_valid, file_path_format

  # Source: US-11 Scenario 2
  Scenario: Missing base schema causes fallback
    Given the base schema file is missing
    When an individual schema tries to reference it
    Then the agent falls back to inline validation rules
    And it logs a warning

Feature: Pattern File Propagation (C11)

  # Source: US-12 Scenario 1
  Scenario: Mid-cycle pattern update detected and reloaded
    Given the Review Coordinator adds a pattern and increments patterns_version
    When the Coder coordinator dispatches the next skill
    Then it detects the version change
    And it reloads patterns before dispatch

  # Source: US-12 Scenario 2
  Scenario: Unreadable patterns file uses cached version
    Given the patterns file is unreadable on re-check
    When the coordinator tries to reload
    Then it uses the last successfully read patterns
    And it logs a warning

Feature: Contract Validation Pilot (C12)

  # Source: US-13 Scenario 1
  Scenario: Test document covers three Coder capabilities
    Given the test document exists at .sdd/tests/contract-validation-pilot.md
    When a maintainer reads it
    Then they find scenarios for contract-first implementation, coverage enforcement, and debug retry loops

  # Source: US-13 Scenario 2
  Scenario: No test codebase records limitation
    Given no suitable test codebase is available
    When the test document is created
    Then the results section states "Not yet executed" with a reason
```

### 11.3 Integration Tests (Cross-File Consistency)

Since this system is file-based, "integration tests" verify consistency across related files:

| Boundary | Files | Verification | Method |
|----------|-------|-------------|--------|
| Enum registry -> Agent files | enums.yaml, all *.agent.md | Enum values referenced in agents match registry values | grep + compare |
| Enum registry -> Schemas | enums.yaml, all *.schema.yaml | Enum values in schema validation rules match registry | YAML parse + compare |
| Base schema -> Individual schemas | base-handoff.schema.yaml, individual schemas | `base_schema` references resolve correctly | YAML parse |
| Agent error policy -> Architecture doc | *.agent.md, architecture.md | Every agent referenced in architecture doc matches actual agent files | Cross-file search |
| WP frontmatter fields -> Agent instructions | All *.agent.md, SKILL.md files | New fields (review_cycles, docs_completed, coverage_*) are read/written as specified | Content review |
| Return schema context fields -> Agent instructions | Return schemas, agent files | Context fields in schemas match what agents produce | Cross-file review |
| RACI labels -> Developer guide | coder.agent.md, review-coordinator.agent.md, developer-guide.md | RACI descriptions are consistent across all three files | Content comparison |

### 11.4 End-to-End Tests (Pipeline Runs)

| Journey | Steps | Success Criteria |
|---------|-------|-----------------|
| Review cycle with escalation | 1. Create WP. 2. Run Coder. 3. Run Reviewer (changes required) x3. 4. Verify Orchestrator escalates. | `review_cycles: 3` in frontmatter; Orchestrator presents escalation to user. |
| Documentation tracking | 1. Create WP. 2. Run through Coder + Reviewer (approved). 3. Run Docs Agent. 4. Verify Orchestrator skips re-documentation. | `docs_completed: true` in WP frontmatter; Orchestrator advances to next WP. |
| Coverage threshold override | 1. Create WP with `coverage_code: 60`. 2. Run Coder. 3. Verify test tool uses 60% threshold. | Coverage tool log shows 60% threshold, not 80%. |
| Dependency-aware ordering | 1. Create WP01 (depends on WP02) and WP02 (no deps). 2. Run Orchestrator. 3. Verify WP02 selected first. | Orchestrator activity log shows WP02 selected before WP01. |

### 11.5 Performance Tests

| Scenario | Operation | Target | Method |
|----------|---------|---------|----|
| Agent startup with enum registry | Additional file reads at startup | <= 3 additional file reads | Count reads in agent log |
| Topological sort with 20 WPs | WP selection delay | Imperceptible (< 1 second) | Manual observation |
| Pattern version check before dispatch | File read overhead per skill dispatch | <= 1 additional file read | Count reads |

### 11.6 Security Tests

| Test | OWASP Category | Verification |
|------|---------------|-------------|
| No secrets in new files | A02 Cryptographic Failures | grep all new/modified files for common secret patterns (API_KEY, TOKEN, SECRET, password) |
| Path validation in return schemas | A01 Broken Access Control | Verify schema path patterns reject paths outside `.sdd/plans/` |
| No stack traces in Activity Logs | A09 Logging Failures | grep Activity Log entries for stack trace patterns |
| Enum registry integrity | A08 Data Integrity | Verify enums.yaml content matches spec definitions exactly |

---

## 14. Open Questions

No open questions. All decisions resolved during specification.

---

## 15. Glossary

| Term | Definition |
|------|-----------|
| Acceptance Criteria | Checkable conditions in a WP that define when a task is complete |
| Activity Log | A timestamped log section within each WP file recording agent actions |
| Agent | An LLM-based actor in the SDD pipeline with a specific role (e.g., Coder, Reviewer) |
| Base Schema | A shared YAML file defining reusable validation patterns referenced by individual handoff schemas |
| BDD | Behavior-Driven Development -- a testing approach using Given/When/Then scenarios |
| Breaking Change | A schema modification that removes or alters existing required fields, types, or enum values |
| Canonical Format | The single authoritative format definition for Activity Log entries |
| Companion Artifact | A typed YAML or code file generated alongside a spec as a machine-readable contract |
| Coordinator Agent | An agent that dispatches skill subagents (Spec Architect, Planner, Coder, Docs Agent) |
| Critical-Path Agent | An agent whose failure halts the pipeline (Spec Architect, Planner, Coder) |
| Advisory Agent | An agent whose failure does not halt the pipeline (Review Coordinator, Docs Agent) |
| Dependency Graph | A directed acyclic graph of WP dependencies derived from `depends_on` frontmatter fields |
| Enum Registry | The central YAML file (`.github/schemas/enums.yaml`) defining all valid enumeration values |
| Escalation | The Orchestrator presenting a WP to the user after exceeding the review cycle threshold |
| FR | Functional Requirement -- a mandatory behavior described with SHALL/SHALL NOT |
| Frontmatter | YAML metadata at the top of a markdown file delimited by `---` |
| Handoff | A structured signal from one agent to another, validated against a schema |
| Handoff Schema | A YAML file defining required fields, types, and validation rules for an agent-to-agent handoff |
| Lane | The workflow state of a WP: planned, doing, for_review, to_do, done, or blocked |
| Maker/Checker | A quality pattern where the Coder (maker) checks criteria and the Reviewer (checker) verifies them |
| NFR | Non-Functional Requirement -- a quality attribute (performance, security, scalability) |
| Orchestrator | The coordinating agent that selects WPs, invokes agents, and manages pipeline flow |
| Pattern File | A markdown file tracking recurring mistakes and lessons learned, consumed by coordinator agents |
| patterns_version | An integer in pattern file frontmatter incremented when patterns are added, modified, or retired |
| Pipeline Stage | The overall state of the SDD pipeline: idle, ideation, specification, planning, implementation, review, documentation, complete |
| RACI | Responsible, Accountable, Consulted, Informed -- a responsibility assignment matrix |
| Return Handoff | A completion signal from a downstream agent back to the Orchestrator |
| Review Cycle | One round of Coder implementation followed by Review Coordinator evaluation |
| review_cycles | An integer WP frontmatter field tracking the number of review cycles |
| docs_completed | A boolean WP frontmatter field indicating whether documentation has been generated |
| coverage_code | An optional integer (0-100) WP frontmatter field overriding the default code coverage threshold |
| coverage_branch | An optional integer (0-100) WP frontmatter field overriding the default branch coverage threshold |
| SC | Success Criterion -- a measurable outcome defining project success |
| SDD | Specification-Driven Development -- the methodology implemented by this pipeline |
| Skill | A specialized instruction set dispatched by a coordinator agent for a specific task |
| Spec Status | The lifecycle state of a specification: Draft, Validated, or Approved |
| Topological Sort | An algorithm that orders WPs so that each WP comes after its dependencies |
| US | User Story -- a feature described from a user's perspective with acceptance scenarios |
| version_history | A YAML array in schema files documenting all changes with version, date, and description |
| WP | Work Package -- a discrete unit of implementation work with frontmatter metadata and tasks |

---

## 16. Traceability Matrix

| FR ID | Requirement Summary | User Story | Acceptance Scenario | Test Type | Test Section Ref |
|-------|-------------------|------------|--------------------|-----------|----|
| FR-001 | WP frontmatter supports review_cycles field | US-01 | Scenario 2 (missing field treated as 0) | content, BDD | 11.1, 11.2 |
| FR-002 | Review Coordinator increments review_cycles | US-01 | Scenario 1 (escalation after 3 cycles) | BDD, E2E | 11.2, 11.4 |
| FR-003 | WP frontmatter supports docs_completed field | US-02 | Scenario 2 (missing field treated as false) | content, BDD | 11.1, 11.2 |
| FR-004 | Docs Agent sets docs_completed: true | US-02 | Scenario 3 (Docs Agent signals completion) | BDD, E2E | 11.2, 11.4 |
| FR-005 | Orchestrator reads review_cycles from frontmatter | US-01 | Scenario 1 (escalation after 3 cycles) | BDD, E2E | 11.2, 11.4 |
| FR-006 | Orchestrator reads docs_completed from frontmatter | US-02 | Scenario 1 (Orchestrator invokes Docs Agent) | BDD, E2E | 11.2, 11.4 |
| FR-007 | Absent fields treated as defaults | US-01, US-02 | US-01 Scenario 2, US-02 Scenario 2 | BDD | 11.2 |
| FR-008 | Central enum registry exists at enums.yaml | US-06 | Scenario 3 (missing file causes halt) | content, BDD | 11.1, 11.2 |
| FR-009 | Enum registry defines 4 enum groups | US-06 | Scenario 1 (contains all lane values) | content | 11.1 |
| FR-010 | Lane enum defines 6 values | US-06 | Scenario 1 (exact lane values) | content, BDD | 11.1, 11.2 |
| FR-011 | Spec status enum defines 3 values | US-06 | Scenario 2 (no "Final" references) | content | 11.1 |
| FR-012 | Pipeline stage enum defines 8 values | US-06 | Scenario 1 | content | 11.1 |
| FR-013 | Review status enum defines 4 values | US-06 | Scenario 1 | content | 11.1 |
| FR-014 | Agent files cite enums.yaml as source | US-06 | Scenario 1 | content | 11.1 |
| FR-015 | Planner removes "Final" spec status | US-06 | Scenario 2 (no "Final" in Planner) | content, BDD | 11.1, 11.2 |
| FR-016 | Canonical Activity Log format | US-07 | Scenario 1, Scenario 2 | BDD | 11.2 |
| FR-017 | Coder uses canonical Activity Log format | US-07 | Scenario 1 (Coder format) | content, BDD | 11.1, 11.2 |
| FR-018 | Review Coordinator uses canonical format | US-07 | Scenario 2 (Review Coordinator format) | content, BDD | 11.1, 11.2 |
| FR-019 | Docs Agent uses canonical format | US-07 | Scenario 2 | content | 11.1 |
| FR-020 | Canonical format documented in enums.yaml | US-07 | Scenario 3 (single authoritative definition) | content, BDD | 11.1, 11.2 |
| FR-021 | Reviewer-to-orchestrator schema exists | US-08 | Scenario 1 (schema file exists) | content, BDD | 11.1, 11.2 |
| FR-022 | Reviewer-to-orchestrator required fields | US-08 | Scenario 2 (required context fields) | content, BDD | 11.1, 11.2 |
| FR-023 | Coder-complete-to-orchestrator schema exists | US-08 | Scenario 1 | content | 11.1 |
| FR-024 | Coder-complete-to-orchestrator required fields | US-08 | Scenario 1 | content | 11.1 |
| FR-025 | Docs-agent-to-orchestrator schema exists | US-08 | Scenario 1 | content | 11.1 |
| FR-026 | Docs-agent-to-orchestrator required fields | US-08 | Scenario 3 (docs context fields) | content, BDD | 11.1, 11.2 |
| FR-027 | Return schemas follow handoff/v1 format | US-08 | Scenario 1 | content, integration | 11.1, 11.3 |
| FR-028 | Architecture doc contains error-handling policy | US-03 | Scenario 1 (subsection exists) | content, BDD | 11.1, 11.2 |
| FR-029 | Agents categorized as critical-path or advisory | US-03 | Scenario 1 (categorization exists) | content, BDD | 11.1, 11.2 |
| FR-030 | Agent files include error policy reference comment | US-03 | Scenario 2 (agent files reference policy) | content, BDD | 11.1, 11.2 |
| FR-031 | Coder labeled as Responsible for acceptance criteria | US-09 | Scenario 1 (Responsible label) | content, BDD | 11.1, 11.2 |
| FR-032 | Reviewer labeled as Accountable/Verifier | US-09 | Scenario 1 | content, BDD | 11.1, 11.2 |
| FR-033 | Developer guide documents maker/checker pattern | US-09 | Scenario 2 (developer guide section) | content, BDD | 11.1, 11.2 |
| FR-034 | WP frontmatter supports coverage_code field | US-04 | Scenario 3 (out-of-range causes halt) | content, BDD | 11.1, 11.2 |
| FR-035 | WP frontmatter supports coverage_branch field | US-04 | Scenario 3 | content | 11.1 |
| FR-036 | code-unit-tests reads coverage thresholds | US-04 | Scenario 1 (WP-specific thresholds) | BDD, E2E | 11.2, 11.4 |
| FR-037 | Defaults to 80/90 when absent | US-04 | Scenario 2 (defaults used) | BDD | 11.2 |
| FR-038 | code-env-setup reads coverage thresholds | US-04 | Scenario 1 | content | 11.1 |
| FR-039 | spec-test-strategy references configurable thresholds | US-04 | Scenario 1 | content | 11.1 |
| FR-040 | Orchestrator uses topological sort for WP selection | US-05 | Scenario 1 (dependency-aware selection) | BDD, E2E | 11.2, 11.4 |
| FR-041 | Lowest WP number as tiebreaker | US-05 | Edge Case 1 (no deps, lowest number wins) | BDD | 11.2 |
| FR-042 | Circular dependency detection | US-05 | Scenario 2 (circular dependency halt) | BDD | 11.2 |
| FR-043 | Missing depends_on treated as no dependencies | US-05 | Edge Case 1 | BDD | 11.2 |
| FR-044 | Schema files include version_history | US-10 | Scenario 1 (version_history exists) | content, BDD | 11.1, 11.2 |
| FR-045 | Version history entry has version, date, description | US-10 | Scenario 1 | content | 11.1 |
| FR-046 | Breaking changes increment version | US-10 | Scenario 2 (breaking change) | BDD | 11.2 |
| FR-047 | Additive changes keep version | US-10 | Scenario 3 (additive change) | BDD | 11.2 |
| FR-048 | Schema validates path placeholders | US-10 | Scenario 1 | content | 11.1 |
| FR-049 | Shared base schema exists | US-11 | Scenario 1 (base schema exists) | content, BDD | 11.1, 11.2 |
| FR-050 | Base schema defines 3 validation patterns | US-11 | Scenario 1 (reusable patterns) | content, BDD | 11.1, 11.2 |
| FR-051 | Individual schemas reference base schema | US-11 | Scenario 1 | content, integration | 11.1, 11.3 |
| FR-052 | Pattern files include patterns_version | US-12 | Scenario 1 (version change detected) | content, BDD | 11.1, 11.2 |
| FR-053 | Review Coordinator increments patterns_version | US-12 | Scenario 1 | BDD | 11.2 |
| FR-054 | Coordinators re-read on version change | US-12 | Scenario 1 (reload on change), Scenario 2 (fallback) | BDD | 11.2 |
| FR-055 | Contract validation pilot document exists | US-13 | Scenario 1 (test document exists) | content, BDD | 11.1, 11.2 |
| FR-056 | Pilot exercises 3 Coder capabilities | US-13 | Scenario 1 (three capabilities) | content, BDD | 11.1, 11.2 |
| FR-057 | Pilot records categorized results | US-13 | Scenario 2 (limitation recorded) | BDD | 11.2 |

### Traceability Summary

- Total FRs: 57
- Total USes: 13
- Total Gherkin scenarios: 38
- Matrix rows: 57
- Traceability gaps: 0 (complete)
- Orphan FRs: 0
- Orphan USes: 0
- Orphan tests: 0

---

## 17. Technical References

### Standards & Conventions
- RFC 2119 -- Key words for use in RFCs to Indicate Requirement Levels. https://datatracker.ietf.org/doc/html/rfc2119

### Security
- OWASP Top 10 (2021). https://owasp.org/www-project-top-ten/
- OWASP Secure Coding Practices Quick Reference Guide. https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/

### Architecture Patterns
- Topological sorting for dependency resolution. Standard graph algorithm (Kahn's algorithm or DFS-based).
- RACI Matrix. Standard responsibility assignment framework.

### Pipeline-Specific
- GitHub Copilot Chat Custom Instructions. https://docs.github.com/en/copilot/customizing-copilot/adding-repository-custom-instructions-for-github-copilot
- YAML 1.2 Specification. https://yaml.org/spec/1.2.2/

---

## 18. Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-07-10 | Spec Architect | Initial specification |

---
