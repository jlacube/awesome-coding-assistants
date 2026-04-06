---
lane: doing
---

# WP41 - WP Frontmatter Extensions

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/010-sdd-pipeline-hardening.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP40 |
| Goal | Add review_cycles and docs_completed frontmatter fields to WP files and update agents to read/write them instead of parsing Activity Logs |
| Status | Not Started |
| Independent Test | Create a WP with `review_cycles: 3` and `docs_completed: true` in frontmatter. Verify the Orchestrator triggers escalation and skips Docs Agent invocation without any Activity Log entries. |
| Parallelisable | Yes (with WP42, WP43) |
| Prompt | `.sdd/plans/WP41-wp-frontmatter-extensions.md` |

## Objective

Extend WP file YAML frontmatter with two new optional fields -- `review_cycles` (integer, default 0) and `docs_completed` (boolean, default false) -- and update the Orchestrator, Review Coordinator, and Docs Agent to read/write these fields instead of scanning Activity Log text. This replaces fragile text parsing with structured, machine-readable state, while maintaining backward compatibility with existing WP files.

## Spec References

FR-001, FR-002, FR-003, FR-004, FR-005, FR-006, FR-007, Section 7.1 (WP Frontmatter Extended), Section 8.1 (WP Frontmatter Read/Write Interface), Section 6.1 (Review Cycle Tracking), Section 6.2 (Documentation Completion Tracking), US-01, US-02

## Tasks

### T41-01 - Define review_cycles frontmatter field

- **Description**: Update the WP frontmatter schema documentation to include `review_cycles` as an optional integer field with default 0. Ensure all agents that read WP frontmatter understand this field.
- **Spec refs**: FR-001, FR-007, Section 7.1
- **Parallel**: Yes (with T41-02)
- **Acceptance criteria**:
  - [x] WP frontmatter schema accepts `review_cycles` as a valid optional field of type integer (FR-001)
  - [x] If `review_cycles` is present but not a non-negative integer, the reading agent SHALL treat it as 0 and log a warning (FR-001)
  - [x] Agents reading WP files created before this change (no `review_cycles` field) SHALL treat the absent field as 0 (FR-007)
- **Test requirements**: content, BDD (US-01 Scenario 2, Scenario 3)
- **Depends on**: none
- **Implementation Guidance**:
  - This is a schema definition task -- update any WP template documentation to show the new field
  - Error E-003 (INVALID_FIELD_TYPE): log warning and use default when type is wrong
  - Error E-006 (FIELD_ABSENT): use default 0, no error
  - Files to create/modify: WP template documentation or shared schema reference

### T41-02 - Define docs_completed frontmatter field

- **Description**: Update the WP frontmatter schema documentation to include `docs_completed` as an optional boolean field with default false.
- **Spec refs**: FR-003, FR-007, Section 7.1
- **Parallel**: Yes (with T41-01)
- **Acceptance criteria**:
  - [x] WP frontmatter schema accepts `docs_completed` as a valid optional field of type boolean (FR-003)
  - [x] If `docs_completed` is present but not a boolean, the reading agent SHALL treat it as false and log a warning (FR-003)
  - [x] Agents reading WP files created before this change (no `docs_completed` field) SHALL treat the absent field as false (FR-007)
- **Test requirements**: content, BDD (US-02 Scenario 2, Edge Case: string "true")
- **Depends on**: none
- **Implementation Guidance**:
  - Same approach as T41-01 for docs_completed
  - Error E-003: non-boolean `docs_completed` value -> treat as false, log warning
  - Error E-006: absent field -> treat as false

### T41-03 - Update Review Coordinator to increment review_cycles

- **Description**: Add logic to the Review Coordinator agent instructions so that when it sets a WP's lane to `to_do` (rework requested), it also increments `review_cycles` by 1 in the WP frontmatter.
- **Spec refs**: FR-002, Section 6.1 (step 1), Section 7.1 (state machine: for_review -> to_do)
- **Parallel**: No (modifies review-coordinator.agent.md)
- **Acceptance criteria**:
  - [x] Review Coordinator SHALL increment `review_cycles` by 1 in WP frontmatter each time it sets lane to `to_do` (FR-002)
  - [x] If `review_cycles` field is absent, the Review Coordinator SHALL add it with value 1 (FR-002)
  - [x] The increment and lane change happen together as a single frontmatter update
  - [x] Given a WP with `review_cycles: 2` and verdict "Changes Required", when the Review Coordinator updates, then `review_cycles` becomes 3 (US-01 Scenario 1)
- **Test requirements**: BDD (US-01 Scenario 1)
- **Depends on**: T41-01
- **Implementation Guidance**:
  - Locate the section in review-coordinator.agent.md where it handles "Changes Required" verdicts
  - Add frontmatter update instruction: set `lane: to_do` AND `review_cycles: review_cycles + 1`
  - If the field does not exist, add it with value 1 (not 0+1, since this is the first rework)
  - Files to modify: `.github/agents/review-coordinator.agent.md`

### T41-04 - Update Docs Agent to set docs_completed

- **Description**: Add logic to the Docs Agent instructions so that upon successful completion of documentation generation, it sets `docs_completed: true` in the WP frontmatter.
- **Spec refs**: FR-004, Section 6.2 (step 5)
- **Parallel**: No (modifies docs-agent.agent.md)
- **Acceptance criteria**:
  - [ ] Docs Agent SHALL set `docs_completed: true` in WP frontmatter upon successful completion (FR-004)
  - [ ] If the WP file cannot be written, the Docs Agent SHALL log the error and report it in its completion signal (FR-004)
  - [ ] Given the Docs Agent finishes documentation, then `docs_completed: true` is set in frontmatter (US-02 Scenario 3)
- **Test requirements**: BDD (US-02 Scenario 3)
- **Depends on**: T41-02
- **Implementation Guidance**:
  - Locate the completion/commit section of docs-agent.agent.md
  - Add frontmatter write instruction before the completion signal
  - Error E-041 (WP_FILE_WRITE_FAILURE): Docs Agent is advisory, so log error but continue
  - Files to modify: `.github/agents/docs-agent.agent.md`

### T41-05 - Update Orchestrator to read review_cycles from frontmatter

- **Description**: Replace the Orchestrator's Activity Log scanning approach for review cycle counting with a direct read of `review_cycles` from WP frontmatter. Use `review_cycles >= 3` for escalation decisions.
- **Spec refs**: FR-005, Section 6.1 (steps 3-5)
- **Parallel**: No (modifies orchestrator.agent.md)
- **Acceptance criteria**:
  - [ ] Orchestrator SHALL read `review_cycles` from WP frontmatter to count review cycles (FR-005)
  - [ ] Escalation decision is based on `review_cycles >= 3` from frontmatter, not Activity Log entry counting (FR-005)
  - [ ] If `review_cycles` is absent or unparseable, the Orchestrator SHALL treat it as 0 (FR-005)
  - [ ] No Activity Log scanning logic remains for review_cycles determination
- **Test requirements**: BDD (US-01 Scenario 1, Scenario 2), E2E (Section 11.4 row 1)
- **Depends on**: T41-01
- **Implementation Guidance**:
  - Search for any Activity Log parsing logic related to review cycle counting in orchestrator.agent.md
  - Replace with: "Read `review_cycles` from WP frontmatter. If absent or invalid, treat as 0."
  - Escalation threshold: `review_cycles >= 3` (existing behavior, just different data source)
  - Files to modify: `.github/agents/orchestrator.agent.md`

### T41-06 - Update Orchestrator to read docs_completed from frontmatter

- **Description**: Replace the Orchestrator's Activity Log scanning approach for documentation status with a direct read of `docs_completed` from WP frontmatter.
- **Spec refs**: FR-006, Section 6.2 (steps 1-2)
- **Parallel**: No (modifies orchestrator.agent.md, sequential with T41-05)
- **Acceptance criteria**:
  - [ ] Orchestrator SHALL read `docs_completed` from WP frontmatter to determine documentation status (FR-006)
  - [ ] Documentation status decision is based on `docs_completed == true` from frontmatter (FR-006)
  - [ ] If `docs_completed` is absent or unparseable, the Orchestrator SHALL treat it as false (FR-006)
  - [ ] Given a WP with `lane: done` and `docs_completed: false`, the Orchestrator invokes the Docs Agent (US-02 Scenario 1)
- **Test requirements**: BDD (US-02 Scenario 1, Scenario 2), E2E (Section 11.4 row 2)
- **Depends on**: T41-05
- **Implementation Guidance**:
  - Search for any Activity Log parsing logic related to documentation status in orchestrator.agent.md
  - Replace with: "Read `docs_completed` from WP frontmatter. If absent or invalid, treat as false."
  - Decision: if `docs_completed` is true, skip Docs Agent invocation
  - Files to modify: `.github/agents/orchestrator.agent.md`

### T41-07 - Verify backward compatibility

- **Description**: Verify that all agents can process existing WP files (from WP01-WP39) that lack the new frontmatter fields without errors. Confirm default-on-absent behavior for both fields.
- **Spec refs**: FR-007, NFR-010
- **Parallel**: No (verification task)
- **Acceptance criteria**:
  - [ ] All agents reading `review_cycles` or `docs_completed` SHALL treat absent fields as their default values (0 and false) (FR-007)
  - [ ] WP files created before this hardening pass are processed without errors by all agents (NFR-010)
  - [ ] No agent halts or errors on missing new frontmatter fields
- **Test requirements**: content (review of agent instructions for default handling)
- **Depends on**: T41-03, T41-04, T41-05, T41-06
- **Implementation Guidance**:
  - Review each modified agent file to confirm absent-field handling is explicit
  - Check that no agent has a required field check for review_cycles or docs_completed
  - Cross-reference with Section 8.1 error table: E-006 (FIELD_ABSENT) -> use default, no error

## Implementation Notes

- All deliverables are markdown agent instruction files -- no executable code
- The core change is replacing Activity Log text parsing with structured YAML frontmatter reads
- Both frontmatter fields are opt-in additions (FR-007): existing WP files remain valid
- Review Coordinator performs a dual-write: frontmatter update + Activity Log entry (Design Decision 1, Section 9.4)
- Docs Agent is an advisory agent (best-effort on write failure per C5)
- The Orchestrator modifications (T41-05, T41-06) should be done sequentially since both edit the same file section

## Risks & Mitigations

- **Risk**: Existing Activity Log scanning logic may be deeply embedded in Orchestrator instructions. **Mitigation**: Search for all Activity Log references before editing. Replace incrementally.
- **Risk**: Review Coordinator may have multiple code paths for setting lane=to_do. **Mitigation**: Search for all `to_do` lane changes and ensure each one increments review_cycles.

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
