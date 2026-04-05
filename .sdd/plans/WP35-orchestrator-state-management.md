---
lane: for_review
---

# WP35 - Orchestrator V2: State File Management & Verification

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/008-orchestrator-v2.spec.md` |
| Priority | P0 |
| Lane | planned |
| Depends on | none |
| Goal | Enable cross-session pipeline state persistence via `.sdd/state.md` with WP frontmatter verification |
| Status | Not Started |
| Independent Test | Create `.sdd/state.md`, manually modify WP frontmatter to create discrepancy, invoke Orchestrator -- verify it detects and resolves the discrepancy |
| Parallelisable | No |
| Prompt | `.sdd/plans/WP35-orchestrator-state-management.md` |

## Objective

Implement the persistent state file infrastructure that enables the Orchestrator to track pipeline progress across VS Code sessions. This WP creates the `.sdd/state.md` schema definition, initialization, update protocol, cross-verification with WP frontmatter, valid state transitions, and the read-only constraint on WP frontmatter. It is the foundation for all other Orchestrator V2 features (error recovery, Docs Agent integration, reporting) which depend on the state file.

## Spec References

FR-001, FR-002, FR-003, FR-004, FR-005, Section 7.0 (State Transitions), Section 7.1 (State File Entity), Section 7.2 (ErrorEntry), Section 9.1 (System Design), Section 9.2 (Decision 1: dual verification)

## Tasks

### T35-01 - Define state file schema in orchestrator.agent.md

- **Description**: Add the `.sdd/state.md` YAML frontmatter schema definition to the orchestrator.agent.md prompt. Include all 8 fields (pipeline_stage, current_spec, current_wp, last_agent, last_result, retry_count, error_log, updated_at) with their types, default values, and validation rules. Include the example YAML block from the spec's Section 4.1 Implementation Contract.
- **Spec refs**: FR-001, Section 7.1, Section 7.2
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The Orchestrator SHALL maintain a persistent state file at `.sdd/state.md` with YAML frontmatter containing: pipeline_stage, current_spec, current_wp, last_agent, last_result, retry_count, error_log, updated_at (FR-001)
  - [x] pipeline_stage SHALL be one of: idle, ideation, specification, planning, implementation, review, documentation, complete (FR-001)
  - [x] error_log SHALL be an array of ErrorEntry objects, each with: agent (string), wp (string or null), error_summary (string, 1-500 chars), timestamp (ISO 8601) (Section 7.2)
  - [x] error_log SHALL contain max 50 entries with oldest pruned when exceeded (Section 7.1)
- **Test requirements**: BDD (US-01 Scenario 1)
- **Depends on**: none
- **Implementation Guidance**:
  - Files to create/modify: `.github/agents/orchestrator.agent.md` -- add `<state_schema>` section
  - Patterns: Copy the YAML example block from Section 4.1 Implementation Contract verbatim
  - Known pitfalls: Ensure field types match companion artifact `data-models.ts` exactly -- PipelineStage enum, AgentResult type, ErrorEntry interface
  - Spec validation rules: pipeline_stage enum values, retry_count >= 0, error_log max 50 entries, current_wp format WP followed by 2 digits or null

### T35-02 - Write state file creation logic

- **Description**: Add instructions to orchestrator.agent.md for creating `.sdd/state.md` when it does not exist. Initialize all fields to their default values (pipeline_stage: idle, all nullable fields: null, retry_count: 0, error_log: [], updated_at: current timestamp). Include error handling for filesystem permission failures.
- **Spec refs**: FR-002
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The Orchestrator SHALL create `.sdd/state.md` if it does not exist, initializing all fields to their defaults (FR-002)
  - [x] If the file cannot be created (e.g., filesystem permission error), halt and report "Cannot create state file at .sdd/state.md" (FR-002)
  - [x] Default values: pipeline_stage=idle, current_spec=null, current_wp=null, last_agent=null, last_result=null, retry_count=0, error_log=[], updated_at=creation time (Section 7.1)
- **Test requirements**: BDD (US-01 Scenario 1)
- **Depends on**: T35-01
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- add state file creation to startup protocol
  - Patterns: Check file existence with read_file, create with create_file if missing
  - Known pitfalls: The `.sdd/` directory must exist as a precondition (FR-001). Do not create the directory -- only the state file.
  - Error handling: If create_file fails, halt immediately with the error message from FR-002

### T35-03 - Write state file update protocol

- **Description**: Add instructions to orchestrator.agent.md for updating `.sdd/state.md` after every agent invocation. The update SHALL record the agent result (success/failed/escalated), update pipeline_stage, current_wp, last_agent, last_result, retry_count, and updated_at. Include error handling for write failures.
- **Spec refs**: FR-003
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The Orchestrator SHALL update `.sdd/state.md` after every agent invocation, recording the result before deciding the next action (FR-003)
  - [x] If the state file cannot be updated, halt and report with the last known state and the update that failed (FR-003)
  - [x] updated_at SHALL be set to current ISO 8601 timestamp on every update (Section 7.1)
- **Test requirements**: BDD (US-01 Scenario 1)
- **Depends on**: T35-01
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- add state update step after each agent delegation
  - Patterns: Read current state file, modify fields, write entire file back. Use replace_string_in_file for the YAML frontmatter block.
  - Known pitfalls: Always update state BEFORE deciding the next action. This is the core sequential invariant.
  - Error handling: On write failure, include both the current state and the intended update in the halt message

### T35-04 - Write state verification logic

- **Description**: Add startup cross-verification protocol to orchestrator.agent.md. On every startup, the Orchestrator reads `.sdd/state.md` and all WP files' frontmatter (lane values), compares them, and resolves discrepancies by trusting WP frontmatter as ground truth. Log any discrepancy found.
- **Spec refs**: FR-004, Section 9.2 (Decision 1)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] On startup, the Orchestrator SHALL cross-verify `.sdd/state.md` against actual WP frontmatter (FR-004)
  - [x] Read state file's current_wp and pipeline_stage, read all WP files' lane values (FR-004 step 1-2)
  - [x] If state file claims current_wp: WP03 with pipeline_stage: review but WP03's lane is done, trust WP frontmatter and update state file accordingly (FR-004 step 3)
  - [x] Log any discrepancy found (FR-004 step 4)
  - [x] Given .sdd/state.md says current_wp=WP03 and pipeline_stage=review, and WP03 frontmatter has lane=done, when the Orchestrator starts, then it updates state.md to reflect lane=done and proceeds to documentation for WP03 (US-01 Scenario 2)
- **Test requirements**: BDD (US-01 Scenario 1, 2)
- **Depends on**: T35-01, T35-02
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- add verification protocol to `<state_machine>` section
  - Patterns: Read state file first, then glob for WP files, read each WP frontmatter, compare lane values against pipeline_stage
  - Known pitfalls: WP frontmatter is ground truth because specialist agents modify it directly. State file is a convenience index that can become stale.
  - Official docs: Section 9.2 Decision 1 explains the rationale for dual verification

### T35-05 - Write state transition validation

- **Description**: Add the valid state transition table from Section 7.0 to the orchestrator prompt. Include validation logic that prevents the Orchestrator from setting pipeline_stage to a value not reachable from the current value. Reference the companion artifact `state-machines.ts` for transition rules.
- **Spec refs**: FR-001 (transitions), Section 7.0
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] The Orchestrator SHALL NOT set pipeline_stage to a value that is not reachable from the current value (Section 7.0)
  - [x] Valid transitions match the state transition table in Section 7.0 exactly (13 valid transitions)
  - [x] Any transition not in the table is invalid (Section 7.0)
  - [x] Transitions match VALID_PIPELINE_TRANSITIONS in companion artifact state-machines.ts
- **Test requirements**: BDD
- **Depends on**: T35-01
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- add state transition table
  - Patterns: Include the full transition table from Section 7.0 as a reference. Before setting pipeline_stage, check the transition is valid.
  - Known pitfalls: The "idle" state has 4 valid transitions (to ideation, specification, planning, implementation) because the Orchestrator can resume at any stage on startup. "complete" has no outgoing transitions.
  - Spec validation rules: 13 valid transitions defined in Section 7.0. implementation->complete and documentation->complete are both valid paths.

### T35-06 - Add WP frontmatter read-only constraint

- **Description**: Add an explicit rule to orchestrator.agent.md that the Orchestrator SHALL NOT modify WP frontmatter directly. Document that only Coder (sets lane=doing, for_review) and Review Coordinator (sets lane=done, to_do) modify WP frontmatter.
- **Spec refs**: FR-005
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] The Orchestrator SHALL NOT modify WP frontmatter directly (FR-005)
  - [x] Only Coder (sets lane=doing, for_review) and Review Coordinator (sets lane=done, to_do) modify WP frontmatter (FR-005)
  - [x] This constraint is documented in the Orchestrator's rules section
- **Test requirements**: none
- **Depends on**: none
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- reinforce existing rule in `<rules>` section
  - Patterns: The existing V1 has a similar rule: "NEVER modify .sdd/ file frontmatter (lane, review_status, etc.) directly". Update to be more specific per FR-005.
  - Known pitfalls: The Orchestrator DOES modify .sdd/state.md (its own state file). The constraint is specifically about WP file frontmatter.

### T35-07 - Verify state schema against companion artifacts

- **Description**: Cross-verify the state file schema defined in the orchestrator prompt against the companion artifacts data-models.ts and state-machines.ts. Ensure field names, types, enum values, and transition rules are identical. Fix any discrepancies.
- **Spec refs**: FR-001, FR-004, Section 7.0, Section 7.1, Section 7.2
- **Parallel**: No
- **Acceptance criteria**:
  - [x] State file schema in orchestrator.agent.md matches PipelineState interface in data-models.ts (all 8 fields, identical names and types)
  - [x] PipelineStage enum values match between orchestrator prompt and data-models.ts
  - [x] State transitions in orchestrator prompt match VALID_PIPELINE_TRANSITIONS in state-machines.ts
  - [x] ErrorEntry fields in orchestrator prompt match ErrorEntry interface in data-models.ts
  - [x] MAX_RETRY_COUNT (2) and MAX_REVIEW_CYCLES (3) match between orchestrator prompt and state-machines.ts
- **Test requirements**: none
- **Depends on**: T35-01, T35-05
- **Implementation Guidance**:
  - Files to read: `.sdd/specs/artifacts/008-orchestrator-v2/data-models.ts`, `.sdd/specs/artifacts/008-orchestrator-v2/state-machines.ts`
  - Files to verify: `.github/agents/orchestrator.agent.md`
  - Patterns: Side-by-side comparison of schema definitions
  - Known pitfalls: The companion artifacts use TypeScript types. The orchestrator prompt uses YAML schema definition. Ensure semantic equivalence, not syntactic identity.

## Implementation Notes

- The orchestrator is a single `.agent.md` file at `.github/agents/orchestrator.agent.md`
- The V1 orchestrator already has state assessment, decision table, and workflow sections. V2 adds the `.sdd/state.md` persistence layer on top.
- All changes target the same file. Tasks T35-05 and T35-06 can run in parallel since they modify independent sections.
- The state file uses YAML frontmatter in a markdown file -- the same format used by WP files.
- "Testing" means manually invoking the Orchestrator and verifying behavior matches BDD scenarios from Section 11.2.

## Parallel Opportunities

- T35-05 (state transitions) and T35-06 (read-only constraint) can be worked concurrently -- they modify independent sections of the orchestrator prompt.

## Risks & Mitigations

- **Risk**: State file schema drift from companion artifacts. **Mitigation**: T35-07 verifies consistency as the final task.
- **Risk**: V1 orchestrator logic conflicts with V2 state management. **Mitigation**: V2 is a full rewrite of the relevant sections -- replace rather than patch.

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-06T12:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-06T12:01:00Z - coder - T35-06 - completed - Updated rules section with specific WP frontmatter read-only constraint per FR-005
- 2026-04-06T12:02:00Z - coder - T35-01 - completed - Added state_schema section with all 8 fields, ErrorEntry schema, and constraints
- 2026-04-06T12:03:00Z - coder - T35-05 - completed - Added valid state transition table with all 13 transitions matching state-machines.ts
- 2026-04-06T12:04:00Z - coder - T35-04 - completed - Added state verification protocol to state_machine section
- 2026-04-06T12:05:00Z - coder - T35-02 - completed - Added state file creation logic in workflow Step 1
- 2026-04-06T12:06:00Z - coder - T35-03 - completed - Added state file update protocol in workflow Step 7
- 2026-04-06T12:07:00Z - coder - T35-07 - completed - Cross-verified schema against data-models.ts and state-machines.ts, all fields match
- 2026-04-06T12:08:00Z - coder - lane=for_review - All tasks complete, all acceptance criteria met
