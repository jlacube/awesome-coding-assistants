---
lane: planned
---

# WP37 - Orchestrator V2: Escalation Support & Status Reporting

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/008-orchestrator-v2.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP35, WP36 |
| Goal | Add universal escalation handling and structured status reporting to complete the Orchestrator V2 feature set |
| Status | Not Started |
| Independent Test | Simulate agent escalation, verify user is prompted with context; verify status report displays after agent completion |
| Parallelisable | No |
| Prompt | `.sdd/plans/WP37-orchestrator-escalation-reporting.md` |

## Objective

Complete the Orchestrator V2 by adding universal escalation support (any agent can escalate to the human with the Orchestrator as intermediary) and structured status reporting (pipeline progress visible after every agent completion). Also handles the edge case of corrupted state files. This WP finalizes the Orchestrator V2 feature set.

## Spec References

FR-014, FR-015, FR-016, FR-017, Section 4.6 (Universal Escalation), Section 4.7 (Status Reporting), Section 5 (Edge Cases)

## Tasks

### T37-01 - Write universal escalation support

- **Description**: Add escalation handling to orchestrator.agent.md. When a delegated agent reports an escalation (e.g., spec ambiguity, environment issue, unresolvable conflict), the Orchestrator SHALL record it in the state file (last_result: escalated), present the escalation to the user with full context, and wait for user response before continuing.
- **Spec refs**: FR-014
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] When a delegated agent reports an escalation, the Orchestrator SHALL record it in the state file with last_result: escalated (FR-014 step 1)
  - [ ] The Orchestrator SHALL present the escalation to the user with full context (FR-014 step 2)
  - [ ] The Orchestrator SHALL wait for user response before continuing (FR-014 step 3)
  - [ ] Escalation support applies to ANY agent (Ideation, Spec Architect, Planner, Coder, Review Coordinator, Docs Agent) (FR-014)
- **Test requirements**: BDD
- **Depends on**: none
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- add escalation branch to result processing
  - Patterns: Use vscode_askQuestions to present escalation context and wait for user input
  - Known pitfalls: Distinguish between agent failure (error, no output, timeout) and agent escalation (deliberate report of an unresolvable issue). Both are recorded in state file but handled differently.
  - Error handling: last_result: escalated is distinct from last_result: failed

### T37-02 - Write escalation resolution logic

- **Description**: Add logic for handling user responses to escalations. When the user resolves an escalation, the Orchestrator SHALL determine which agent to re-invoke based on the resolution. This may not be the same agent that escalated.
- **Spec refs**: FR-015
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] When the user resolves an escalation, the Orchestrator SHALL determine which agent to re-invoke based on the resolution (FR-015)
  - [ ] The re-invoked agent is not necessarily the same agent that escalated (FR-015)
  - [ ] The Orchestrator reads the resolution, re-assesses state, and invokes the appropriate agent based on updated understanding
- **Test requirements**: BDD
- **Depends on**: T37-01
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- add resolution handling after escalation
  - Patterns: After user responds, reset last_result to null, re-run the state assessment protocol (like a mini-startup), then use the decision table normally to determine next action
  - Known pitfalls: The user might resolve an escalation by modifying files manually (e.g., fixing a spec ambiguity). The Orchestrator must re-read state from disk, not rely on cached state.
  - Official docs: Section 6.4 (Escalation Flow) shows the user interaction pattern

### T37-03 - Write status report format

- **Description**: Add the structured status report template from Section 4.7 (FR-016). After every agent completion, the Orchestrator SHALL display: last action, result, retries, stage table (showing status for every pipeline stage including per-WP implementation, review, and docs), and next action.
- **Spec refs**: FR-016
- **Parallel**: Yes
- **Acceptance criteria**:
  - [ ] After every agent completion, the Orchestrator SHALL display a status report (FR-016)
  - [ ] Status report includes: last action ({agent} completed {what it did}), result ({success/needs-fixes/blocked/escalated}), retries ({retry_count}/2) (FR-016)
  - [ ] Status report includes a stage table with all pipeline stages and per-WP status (FR-016)
  - [ ] Status report includes next action description (FR-016)
  - [ ] Report format matches the template in Section 4.7 exactly
- **Test requirements**: BDD
- **Depends on**: none
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- update `<output_format>` section
  - Patterns: The V1 already has a status reporting format. V2 adds retry count, Docs status per WP, and the escalated result type.
  - Known pitfalls: The stage table must be dynamic -- show rows for every WP that exists, not just a fixed template. Per-WP rows include: WP{NN} Impl (lane value), WP{NN} Review (passed/failed/pending), WP{NN} Docs (done/pending).

### T37-04 - Write todo list pipeline tracker

- **Description**: Add manage_todo_list (or #tool:todo) usage to orchestrator.agent.md. The Orchestrator SHALL maintain a high-level pipeline tracker visible to the user throughout the session, updating it after every agent completion.
- **Spec refs**: FR-017
- **Parallel**: Yes
- **Acceptance criteria**:
  - [ ] The Orchestrator SHALL use manage_todo_list to maintain a high-level pipeline tracker visible to the user throughout the session (FR-017)
  - [ ] The tracker updates after every agent completion to reflect current pipeline state
  - [ ] The tracker shows each pipeline stage with its current status
- **Test requirements**: BDD
- **Depends on**: none
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- add todo list management to the orchestration loop
  - Patterns: The V1 already references #tool:todo. V2 formalizes this as a required step after every agent completion.
  - Known pitfalls: The todo list is a UI element, not a file. It is managed via the manage_todo_list tool and persists within the session only.

### T37-05 - Write corrupted state file recovery

- **Description**: Add handling for corrupted or invalid YAML in `.sdd/state.md`. When the state file exists but has corrupted or invalid YAML, the Orchestrator SHALL recreate it from WP frontmatter ground truth and log a warning. This covers the edge case from Section 5.
- **Spec refs**: FR-004 (edge case from Section 5)
- **Parallel**: Yes
- **Acceptance criteria**:
  - [ ] When .sdd/state.md has corrupted or invalid YAML, the Orchestrator SHALL recreate it from WP frontmatter ground truth (Edge case, Section 5)
  - [ ] The Orchestrator SHALL log a warning about the corrupted state file
  - [ ] After recreation, the state file accurately reflects the actual state of all WPs
- **Test requirements**: BDD
- **Depends on**: none
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- add corrupted state handling to startup protocol
  - Patterns: During startup state file read, if YAML parsing fails, fall back to scanning WP frontmatter to reconstruct state. Use the same logic as the verification protocol (T35-04) but applied to full reconstruction.
  - Known pitfalls: Corrupted YAML could be partial (some fields readable) or total (unparseable). Treat both cases the same: delete and recreate from WP ground truth.

### T37-06 - Integration verification of full Orchestrator V2

- **Description**: Perform a comprehensive walkthrough of all Orchestrator V2 features working together. Verify: state file creation, state file updates, cross-verification, sequential execution, Docs Agent integration, error retry, max retry escalation, review failure escalation, universal escalation, status reporting, todo list tracking, corrupted state recovery.
- **Spec refs**: FR-001 to FR-017, Section 6.1-6.4 (User Flows), Section 11.2 (BDD Scenarios)
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] State file: created on first run, updated after each agent, cross-verified against WP frontmatter on startup
  - [ ] Sequential execution: one agent at a time, state read after completion, decision made from fresh state
  - [ ] Docs Agent: invoked after WP approval (lane=done), skipped when lane=to_do
  - [ ] Error recovery: retry on failure (retry_count < 2), escalate on max retries (retry_count >= 2)
  - [ ] Review failure: halt after 3 review cycles returning lane=to_do
  - [ ] Escalation: recorded in state file, presented to user, pipeline waits for resolution
  - [ ] Status report: displayed after every agent completion with correct format
  - [ ] Todo list: updated after every agent completion
  - [ ] Corrupted state: recreated from WP frontmatter ground truth
- **Test requirements**: BDD (all Section 11.2 scenarios)
- **Depends on**: T37-01 to T37-05
- **Implementation Guidance**:
  - Verification method: Manual walkthrough against all BDD scenarios from Section 11.2
  - Walk through each user flow (Section 6.1 Full Pipeline, 6.2 Cross-Session Resume, 6.3 Recovery After Failure, 6.4 Escalation Flow)
  - Verify all 17 FRs are implemented correctly
  - Check edge cases from Section 5: corrupted state file, WP with no dependencies, Docs Agent failure

## Implementation Notes

- All changes target `.github/agents/orchestrator.agent.md`. This is the final WP that completes the V2 rewrite.
- Tasks T37-03, T37-04, and T37-05 can run in parallel since they modify independent sections of the orchestrator prompt.
- "Testing" means manually invoking the Orchestrator and verifying behavior matches BDD scenarios from Section 11.2.
- The Orchestrator V2 is fully self-contained in one agent file. No skill files, no separate contract files -- all logic is in the agent prompt.

## Parallel Opportunities

- T37-03 (status report), T37-04 (todo list), and T37-05 (corrupted state) can all be worked concurrently -- they modify independent sections.

## Risks & Mitigations

- **Risk**: Escalation resolution logic is ambiguous -- spec says "determine which agent to re-invoke" but does not specify how. **Mitigation**: Re-use the existing decision table. After resolution, re-assess state and let the decision table determine the next agent.
- **Risk**: Status report format becomes stale as WP count grows. **Mitigation**: Make the report dynamic -- generate rows based on actual WPs discovered from the filesystem.

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
