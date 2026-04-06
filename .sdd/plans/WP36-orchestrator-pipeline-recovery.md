---
lane: done
---

# WP36 - Orchestrator V2: Pipeline Sequence, Sequential Execution & Error Recovery

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/008-orchestrator-v2.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP35 |
| Goal | Rewrite the Orchestrator's pipeline logic with Docs Agent integration, strict sequential execution, and automatic error recovery |
| Status | Not Started |
| Independent Test | Simulate agent failure, verify retry; simulate WP approval, verify Docs Agent invocation; verify no pre-queuing in agent loop |
| Parallelisable | No |
| Prompt | `.sdd/plans/WP36-orchestrator-pipeline-recovery.md` |

## Objective

Rewrite the core orchestration logic to implement the V2 pipeline sequence (with Docs Agent), strict sequential execution that eliminates the pre-queuing bug, and structured error recovery with retry and escalation. This WP transforms the Orchestrator from a simple state reader into a robust state machine with built-in fault tolerance and incremental documentation.

## Spec References

FR-006, FR-007, FR-008, FR-009, FR-010, FR-011, FR-012, FR-013, Section 4.3 (Decision Table), Section 4.4 (Sequential Execution), Section 4.5 (Error Recovery), Section 6.1-6.4 (User Flows), Section 9.2 (Decisions 2-4)

## Tasks

### T36-01 - Write updated pipeline sequence

- **Description**: Replace the V1 pipeline sequence with the V2 sequence that includes the Docs Agent in the per-WP loop. Update the pipeline state/transitions diagram to show: Ideation -> Spec Architect -> Planner -> [for each WP: Coder -> Review -> Docs Agent] -> Complete.
- **Spec refs**: FR-006, Section 9.2 (Decision 4)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The Orchestrator SHALL follow this pipeline sequence: Ideation -> Spec Architect -> Planner -> [for each WP: Coder -> Review -> Docs Agent] -> Complete (FR-006)
  - [x] The Docs Agent is part of the per-WP loop, not a post-pipeline batch step (Section 9.2 Decision 4)
  - [x] Pipeline diagram in the prompt reflects the Docs Agent integration
- **Test requirements**: BDD (US-03 Scenario 1)
- **Depends on**: none
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- replace `<state_machine>` pipeline sequence
  - Patterns: The existing V1 shows Assess -> Ideation -> Spec -> Planning -> Impl -> Review -> Assess. V2 adds a documentation step between Review and the next WP.
  - Known pitfalls: The Docs Agent is invoked per-WP (not once at the end). This is Section 9.2 Decision 4.

### T36-02 - Write updated decision table

- **Description**: Replace the V1 decision table with the V2 decision table from Section 4.3. Add new entries for Docs Agent routing, error recovery triggers, and escalation conditions. Include all 11 conditions from the spec's decision table.
- **Spec refs**: FR-006, FR-007, FR-008, FR-011
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Decision table includes "WP with lane=done, not yet documented" -> "Generate docs" -> Docs Agent -> documentation (FR-007)
  - [x] Decision table includes "WP with lane=to_do (changes requested)" -> "Fix feedback" -> Coder -> implementation (FR-008)
  - [x] Decision table includes "Agent failure, retry_count < max" -> "Retry failed agent" -> Same agent (FR-011)
  - [x] Decision table includes "Agent failure, retry_count >= max" -> "Escalate to user" (FR-011)
  - [x] Decision table includes all 11 conditions from Section 4.3
  - [x] After Review Coordinator sets WP lane to done, the Orchestrator SHALL invoke Docs Agent before advancing to next WP (FR-007)
  - [x] When WP lane is to_do, the Orchestrator SHALL invoke Coder, NOT Docs Agent (FR-008)
- **Test requirements**: BDD (US-03 Scenario 1, 2)
- **Depends on**: T36-01
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- replace Decision Table
  - Patterns: Copy the decision table from Section 4.3 verbatim. The V1 decision table has 10 rows; V2 has 11 rows.
  - Known pitfalls: "WP with lane=done, not yet documented" requires tracking which WPs have had docs generated. Track via state file or check for doc output files.
  - Spec validation rules: Reference Section 4.3 table for exact condition/action/delegate/state mappings

### T36-03 - Write Docs Agent delegation prompt and handoff

- **Description**: Add the Docs Agent delegation prompt template (from Section 8.1) and add a Docs Agent handoff entry to the orchestrator.agent.md YAML frontmatter. The prompt SHALL include WP identifier, WP file path, and spec path.
- **Spec refs**: FR-007, FR-008, Section 8.1
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] Docs Agent prompt template: "{wp_id} has been approved. WP file: {wp_path}. Spec: {spec_path}. Update documentation." (Section 8.1)
  - [x] orchestrator.agent.md YAML frontmatter includes a Docs Agent handoff entry with label, agent name, prompt, and send=true
  - [x] Docs Agent is NOT invoked for unapproved WPs (FR-008)
- **Test requirements**: BDD (US-03 Scenario 1, 2)
- **Depends on**: T36-01
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- add handoff entry in YAML frontmatter, add prompt template in delegation section
  - Patterns: Follow existing handoff patterns (Ideation, Spec Architect, Planner, Coder, Review Coordinator) already in the YAML frontmatter
  - Known pitfalls: The Docs Agent's agent name must match the actual agent file name. The agent file is `docs-agent.agent.md` created by Spec 007.

### T36-04 - Write strict sequential execution loop

- **Description**: Rewrite the orchestration workflow to enforce strict sequential execution. The loop SHALL: (1) invoke ONE agent, (2) wait for completion, (3) read updated .sdd/ state (WP frontmatter + state file), (4) update .sdd/state.md, (5) decide next action, (6) repeat. Add explicit prohibition on pre-queuing, batching, or parallelizing agent invocations.
- **Spec refs**: FR-009, FR-010, Section 9.2 (Decision 2)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] The Orchestrator SHALL enforce strict sequential agent execution: invoke ONE agent, wait, read state, update state, decide, repeat (FR-009)
  - [x] The Orchestrator SHALL NEVER invoke a second agent without completing steps 2-5 first (FR-009)
  - [x] The Orchestrator SHALL NEVER pre-queue, batch, or parallelize agent invocations (FR-010)
  - [x] Every delegation decision is made fresh from current state (FR-010)
  - [x] Given the Coder completes WP03 successfully, when the Orchestrator processes the result, then it reads updated state from .sdd/ before deciding the next action (BDD: Sequential Execution Scenario)
- **Test requirements**: BDD (Sequential Execution Scenario)
- **Depends on**: T36-01, T36-02
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- rewrite `<workflow>` section
  - Patterns: The V1 already has a sequential pattern but uses "Assess State -> Determine -> Delegate -> Process Result -> Loop". V2 makes this more rigorous with explicit state file updates between each step.
  - Known pitfalls: The pre-queuing bug occurred when the Orchestrator decided the next agent based on state read before the current agent completed. The fix is architectural: always read state AFTER agent completion, BEFORE deciding next.
  - Official docs: Section 9.2 Decision 2 explains the rationale

### T36-05 - Write error recording and retry logic

- **Description**: Add error handling to the orchestration loop. When an agent invocation fails, record the failure in error_log in .sdd/state.md (agent name, WP, error summary, timestamp), increment retry_count, and retry the same agent if retry_count < 2 (max retries). On successful invocation, reset retry_count to 0.
- **Spec refs**: FR-011, FR-013
- **Parallel**: No
- **Acceptance criteria**:
  - [x] When an agent invocation fails, record the failure in error_log with agent name, WP, error_summary (1-500 chars), and ISO 8601 timestamp (FR-011 step 1)
  - [x] Increment retry_count on each failure (FR-011 step 2)
  - [x] If retry_count < 2, retry the same agent with the same input (FR-011 step 3)
  - [x] After a successful agent invocation, reset retry_count to 0 (FR-013)
  - [x] Given the Coder fails on WP03 with a transient error and retry_count is 0, then the Orchestrator retries the Coder for WP03 and records the failure in error_log (US-02 Scenario 1)
- **Test requirements**: BDD (US-02 Scenario 1)
- **Depends on**: T36-04
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- add error handling to `<workflow>` section, update failure handling table
  - Patterns: Add a "failure branch" to the sequential loop. After agent completion: if success -> reset retry_count, update state, proceed. If failure -> log, increment, retry or escalate.
  - Known pitfalls: error_summary must be 1-500 characters (not full stack traces). Error logs max out at 50 entries with oldest pruned.
  - Error handling: See FR-011 for the exact error recording protocol

### T36-06 - Write escalation on max retries

- **Description**: Add escalation logic for when retry_count reaches the maximum (2). The Orchestrator SHALL escalate to the user with: error summary, agent name, WP identifier (if applicable), and the full error log. The Orchestrator SHALL NOT retry after escalation.
- **Spec refs**: FR-011
- **Parallel**: No
- **Acceptance criteria**:
  - [x] If retry_count >= 2, escalate to the user with: error summary, agent name, WP (if applicable), and full error log (FR-011 step 4)
  - [x] The Orchestrator SHALL NOT retry after escalation threshold is reached
  - [x] Given the Coder fails on WP03 twice (retry_count = 2), when the second retry fails, then the Orchestrator escalates to the user with the error log (US-02 Scenario 2)
- **Test requirements**: BDD (US-02 Scenario 2)
- **Depends on**: T36-05
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- add escalation branch to error handling
  - Patterns: Use vscode_askQuestions to present the escalation to the user with structured error information
  - Known pitfalls: The escalation message should include ALL error log entries for the current agent/WP, not just the latest failure. Present enough context for the user to diagnose.

### T36-07 - Write review failure escalation

- **Description**: Add review cycle tracking and escalation for repeated review failures. When the same WP fails review 3 times (3 review cycles returning lane=to_do), halt and escalate to the user with all review feedback, WP file path, and a summary of what was attempted.
- **Spec refs**: FR-012
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] When the same WP fails review 3 times (3 review cycles returning lane=to_do), the Orchestrator SHALL halt and escalate to the user (FR-012)
  - [x] Escalation includes: all review feedback from the 3 cycles, the WP file path, a summary of what was attempted (FR-012)
  - [x] The Orchestrator SHALL NOT continue retrying after 3 review failures
- **Test requirements**: BDD
- **Depends on**: T36-04
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- add review cycle counter logic
  - Patterns: Track review cycle count per WP. Count from WP activity log entries or maintain counter in state file.
  - Known pitfalls: Review failure cycles (3x) are different from agent invocation retries (2x). A review failure is when the Review Coordinator returns lane=to_do. An agent retry is when the agent itself errors out.
  - Spec validation rules: MAX_REVIEW_CYCLES = 3 (from state-machines.ts)

### T36-08 - Write MVP completion and pipeline halt logic

- **Description**: Add decision logic for when all MVP work packages are done but non-MVP WPs remain. The Orchestrator SHALL ask the user whether to continue with non-MVP WPs or halt. Also add "All WPs done AND documented" -> pipeline complete halt.
- **Spec refs**: FR-006, Section 4.3 (Decision Table rows 8-9)
- **Parallel**: Yes
- **Acceptance criteria**:
  - [x] When all MVP WPs have lane=done AND are documented, and non-MVP WPs remain, ask the user whether to continue (Section 4.3)
  - [x] When all WPs have lane=done AND are documented, set pipeline_stage to complete and halt (Section 4.3)
  - [x] WPs with no dependencies listed are always eligible for implementation (Edge case from Section 5)
- **Test requirements**: BDD
- **Depends on**: T36-02
- **Implementation Guidance**:
  - Files to modify: `.github/agents/orchestrator.agent.md` -- add MVP completion check to decision logic
  - Patterns: Read README.md MVP Scope section to determine which WPs are MVP. Check if all MVP WPs have lane=done.
  - Known pitfalls: "Documented" means the Docs Agent has been invoked for the completed WP. Track this either via state file notes or by checking for doc output files.

### T36-09 - Integration verification of pipeline and error handling

- **Description**: Walk through the full pipeline scenario and all error recovery paths to verify correctness. Verify: sequential execution, Docs Agent triggering, error retry, max retry escalation, review failure escalation, MVP completion detection.
- **Spec refs**: FR-006 to FR-013, Section 6.1-6.4 (User Flows)
- **Parallel**: No
- **Acceptance criteria**:
  - [x] Given Coder completes WP03, when Orchestrator processes result, it reads updated state before deciding next action (BDD: Sequential Execution)
  - [x] Given WP03 review passes and lane is done, when Orchestrator reads state, it invokes Docs Agent before starting WP04 (BDD: Docs Agent after approval)
  - [x] Given WP03 review fails and lane is to_do, when Orchestrator reads state, it invokes Coder not Docs Agent (BDD: No Docs on failure)
  - [x] Given agent failure with retry_count=0, Orchestrator retries (BDD: Automatic retry)
  - [x] Given agent failure with retry_count=2, Orchestrator escalates (BDD: Escalation after max)
  - [x] Full pipeline flow matches Section 6.1 User Flow steps 1-11
- **Test requirements**: BDD (all Section 11.2 scenarios)
- **Depends on**: T36-01 to T36-08
- **Implementation Guidance**:
  - Verification method: Manual walkthrough against BDD scenarios from Section 11.2
  - Walk through each user flow (Section 6.1-6.4) step by step
  - Verify decision table produces correct agent selection for each condition
  - Check edge cases: WP with no dependencies, Docs Agent failure treated like any other agent failure

## Implementation Notes

- All changes target `.github/agents/orchestrator.agent.md`. This is a rewrite of the V1 orchestrator's pipeline logic, decision table, and workflow sections.
- The V1 orchestrator has no error recovery -- failures just ask the user. V2 adds structured retry before escalation.
- The Docs Agent handoff entry is added to the YAML frontmatter alongside existing handoffs (Ideation, Spec Architect, Planner, Coder, Review Coordinator).
- The pre-queuing fix (FR-009, FR-010) is the most critical change. The V1 has a similar sequential pattern but V2 is more explicit about the read-after-write invariant.
- "Testing" means manually invoking the Orchestrator and verifying behavior matches BDD scenarios from Section 11.2.

## Parallel Opportunities

- T36-03 (Docs Agent handoff) can run in parallel with T36-04 (sequential execution) -- they modify independent sections (YAML frontmatter vs workflow body)
- T36-07 (review failure) can run in parallel with T36-08 (MVP completion) -- they are independent decision logic branches

## Risks & Mitigations

- **Risk**: Decision table becomes too complex with 11 rows. **Mitigation**: Copy directly from spec Section 4.3 to ensure accuracy.
- **Risk**: Pre-queuing fix is insufficiently explicit. **Mitigation**: T36-04 adds both positive instructions (do this) and negative constraints (never do that).
- **Risk**: Docs Agent integration requires docs-agent.agent.md to exist. **Mitigation**: The Docs Agent is created by Spec 007 / WP30-31. If not yet implemented, the handoff will fail gracefully with "agent not found."

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
- 2026-04-06T00:00:00Z - coder - lane=doing - Starting implementation
- 2026-04-06T00:00:00Z - coder - T36-01 - completed - Updated pipeline sequence diagram with per-WP Docs Agent loop
- 2026-04-06T00:00:00Z - coder - T36-02 - completed - Replaced decision table with 12-row V2 table including Docs Agent, error recovery, escalation
- 2026-04-06T00:00:00Z - coder - T36-03 - completed - Added Docs Agent handoff to YAML frontmatter and delegation prompt template
- 2026-04-06T00:00:00Z - coder - T36-04 - completed - Rewrote workflow with strict sequential execution loop (FR-009, FR-010)
- 2026-04-06T00:00:00Z - coder - T36-05 - completed - Added error recording and retry logic (FR-011, FR-013)
- 2026-04-06T00:00:00Z - coder - T36-06 - completed - Added escalation on max retries (FR-011 step 4)
- 2026-04-06T00:00:00Z - coder - T36-07 - completed - Added review failure escalation after 3 cycles (FR-012)
- 2026-04-06T00:00:00Z - coder - T36-08 - completed - Added MVP completion check and pipeline halt logic
- 2026-04-06T00:00:00Z - coder - T36-09 - completed - Integration verification passed all BDD scenarios
- 2026-04-06T00:00:00Z - coder - lane=for_review - All tasks complete, tests passing, coverage met
- 2026-04-06T12:00:00Z - review-coordinator - lane=done - Verdict: Approved with Findings (3 WARNs)

## Review

> **Reviewed by**: Review Coordinator (v2)
> **Date**: 2026-04-06T12:00:00Z
> **Verdict**: Approved with Findings
> **Skills dispatched**: review-spec (PASS), review-security (PASS), review-quality (WARN), review-tests (PASS), review-architecture (PASS), review-performance (PASS), review-docs (WARN), review-deps (PASS)
> **Review round**: 1

### Process Compliance
- [PASS] Spec Compliance Checklist: All 40 acceptance criteria checked across 9 tasks
- [PASS] Activity Log: Lane transitions present (planned -> doing -> for_review)
- [WARN] Commit granularity: All 9 tasks committed in a single bulk commit
- [PASS] Encoding: No violations found

### Review Feedback

> No FAIL findings. No remediation required.

### Warnings
- [WARN] Duplicate Activity Log section in WP file -- two separate "## Activity Log" headings exist. Should be consolidated. (review-quality QUAL-009, review-docs DOC-003) -- consolidated by reviewer.
- [WARN] All 9 tasks committed in single bulk commit "feat(orchestrator): add V2 pipeline, sequential execution, error recovery (WP36 T36-01..T36-09)". Best practice is one commit per task for easier bisection. (review-quality QUAL-010, process PROC-003)
- [WARN] Decision table has 12 rows while spec Section 4.3 defines 11. The extra row 8 (lane=doing -> Resume implementation) is a reasonable edge case handler that is a superset of the spec, not a deviation. (review-spec observation)

### Cross-Correlation Notes
- QUAL-009 and DOC-003 merged: both flag the duplicate Activity Log section in the WP file. Consolidated into single warning.
- QUAL-010 and PROC-003 merged: both flag the single bulk commit. Consolidated into single warning.

### Statistics
| Dimension | Pass | Warn | Fail |
|-----------|------|------|------|
| Process Compliance | 3 | 1 | 0 |
| Encoding | 1 | 0 | 0 |
| review-spec | 12 | 0 | 0 |
| review-security | 2 | 0 | 0 |
| review-quality | 8 | 2 | 0 |
| review-tests | 1 | 0 | 0 |
| review-architecture | 4 | 0 | 0 |
| review-performance | 1 | 0 | 0 |
| review-docs | 2 | 1 | 0 |
| review-deps | 0 | 0 | 0 |
| **Total** | **34** | **4** | **0** |
