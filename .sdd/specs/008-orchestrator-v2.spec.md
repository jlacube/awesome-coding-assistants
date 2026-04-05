# Orchestrator V2 - State Tracking, Error Recovery, and Sequential Execution -- Specification

> **Source brief**: `.sdd/ideas/002-sdd-pipeline-v2-universal-skill-architecture.md`
> **Feature branch**: `008-orchestrator-v2`
> **Status**: Validated
> **Version**: 1.1

---

## 1. Overview

Upgrade the SDD Pipeline Orchestrator with three improvements: (1) a persistent state file (`.sdd/state.md`) combined with WP frontmatter verification for cross-session progress tracking, (2) structured error recovery with retry and escalation for failed agents, and (3) strict sequential execution that fixes the pre-queuing bug by enforcing the invariant "one agent, re-read state, then decide next." Additionally, the V2 Orchestrator integrates the new Docs Agent into the pipeline sequence and supports the universal escalation model where any agent can escalate to the human.

---

## 2. Goals & Success Criteria

- **SC-001**: Pipeline state survives across VS Code sessions. Verified by: closing and reopening VS Code, then invoking the Orchestrator -- it resumes from the correct pipeline stage without re-reading all WP files manually.
- **SC-002**: Failed agent invocations are retried up to the configured limit before escalating. Verified by: a transient agent failure is retried and succeeds on the second attempt.
- **SC-003**: The pre-queuing bug is eliminated. Verified by: the Orchestrator never invokes a second agent before reading the state produced by the first.
- **SC-004**: The Docs Agent is triggered after every WP approval. Verified by: after Review Coordinator sets lane=done, the Orchestrator invokes the Docs Agent before proceeding to the next WP.

---

## 3. Users & Roles

- **Human Developer (primary user)**: Invokes the Orchestrator to drive the pipeline. Receives status reports. Makes decisions at escalation points (spec approval, plan approval, blocker resolution).
- **Pipeline Agents (delegees)**: Ideation, Spec Architect, Planner, Coder, Review Coordinator, Docs Agent -- all invoked by the Orchestrator as subagents.

---

## 4. Functional Requirements

### 4.1 State File Management

- **FR-001**: The Orchestrator SHALL maintain a persistent state file at `.sdd/state.md` with YAML frontmatter containing:
  - `pipeline_stage`: Current stage (one of: `idle`, `ideation`, `specification`, `planning`, `implementation`, `review`, `documentation`, `complete`)
  - `current_spec`: Path to the active spec file (or `null`)
  - `current_wp`: Current WP identifier being processed (or `null`)
  - `last_agent`: Name of the last agent invoked (or `null`)
  - `last_result`: Result of the last agent invocation (`success`, `failed`, `escalated`, or `null`)
  - `retry_count`: Number of retries attempted for the current agent (integer, default 0)
  - `error_log`: Array of error entries (each with `agent`, `wp`, `error_summary`, `timestamp`)
  - `updated_at`: ISO 8601 timestamp of last state update

  Precondition: `.sdd/` directory exists.
  Postcondition: State file reflects the current pipeline position.
  Error: If `.sdd/state.md` cannot be written, halt and report.

- **FR-002**: The Orchestrator SHALL create `.sdd/state.md` if it does not exist, initializing all fields to their defaults.
  Error: If the file cannot be created (e.g., filesystem permission error), halt and report "Cannot create state file at .sdd/state.md".

- **FR-003**: The Orchestrator SHALL update `.sdd/state.md` after every agent invocation, recording the result before deciding the next action.
  Error: If the state file cannot be updated, halt and report with the last known state and the update that failed.

#### Implementation Contract -- State File

**State file schema** (YAML frontmatter in `.sdd/state.md`):

```yaml
---
pipeline_stage: "implementation"        # idle | ideation | specification | planning | implementation | review | documentation | complete
current_spec: ".sdd/specs/001-foo.spec.md"  # string | null
current_wp: "WP01"                      # string | null
last_agent: "4. Coder"                  # string | null
last_result: "success"                  # success | failed | escalated | null
retry_count: 0                          # integer >= 0
error_log:                              # array of objects
  - agent: "4. Coder"
    wp: "WP01"
    error_summary: "Test suite failed with 3 errors"
    timestamp: "2026-04-05T14:32:00Z"
updated_at: "2026-04-05T14:35:00Z"      # ISO 8601
---

# Pipeline State

Human-readable summary of current state for cross-session continuity.
```

**Inputs**: Agent result (success/failure/escalation), WP status changes.
**Outputs**: Updated `.sdd/state.md`.
**Error behaviors**: Write failure - halt with error message.

---

### 4.2 State Verification

- **FR-004**: On startup, the Orchestrator SHALL cross-verify `.sdd/state.md` against actual WP frontmatter:
  1. Read state file's `current_wp` and `pipeline_stage`
  2. Read all WP files' `lane` values
  3. If state file claims `current_wp: WP03` with `pipeline_stage: review` but WP03's lane is `done`, the Orchestrator SHALL trust the WP frontmatter and update the state file accordingly
  4. Log any discrepancy found

  Rationale: WP frontmatter is modified by specialist agents (Coder, Review Coordinator) and represents ground truth. The state file is a convenience index.

- **FR-005**: The Orchestrator SHALL NOT modify WP frontmatter directly. Only Coder (sets lane=doing, for_review) and Review Coordinator (sets lane=done, to_do) modify WP frontmatter.

---

### 4.3 Updated Pipeline Sequence

- **FR-006**: The Orchestrator SHALL follow this updated pipeline sequence that includes the Docs Agent:

  ```
  Ideation -> Spec Architect -> Planner -> [for each WP: Coder -> Review -> Docs Agent] -> Complete
  ```

- **FR-007**: After the Review Coordinator sets a WP's lane to `done`, the Orchestrator SHALL invoke the Docs Agent for that WP before advancing to the next WP.

- **FR-008**: After the Review Coordinator sets a WP's lane to `to_do` (changes requested), the Orchestrator SHALL invoke the Coder to fix the WP. The Docs Agent is NOT invoked for unapproved WPs.

#### Updated Decision Table

| Condition | Action | Delegate To | State After |
|-----------|--------|-------------|-------------|
| No ideas, no specs, no plans | Ask user for intent | User | idle |
| Ideation brief exists without matching spec | Turn brief into spec | Spec Architect | specification |
| Spec exists without work packages | Decompose spec into WPs | Planner | planning |
| WP with lane=planned, dependencies met | Implement WP | Coder | implementation |
| WP with lane=for_review | Review WP | Review Coordinator | review |
| WP with lane=done, not yet documented | Generate docs | Docs Agent | documentation |
| WP with lane=to_do (changes requested) | Fix feedback | Coder | implementation |
| All WPs have lane=done AND documented | Pipeline complete | None (halt) | complete |
| All MVP WPs done, non-MVP remain | Ask user to continue | User | idle |
| Agent failure, retry_count < max | Retry failed agent | Same agent | same |
| Agent failure, retry_count >= max | Escalate to user | User | same |

---

### 4.4 Strict Sequential Execution

- **FR-009**: The Orchestrator SHALL enforce strict sequential agent execution:
  1. Invoke exactly ONE agent
  2. Wait for agent completion
  3. Read updated `.sdd/` state (WP frontmatter + state file)
  4. Update `.sdd/state.md`
  5. Decide next action based on updated state
  6. Repeat

  The Orchestrator SHALL NEVER invoke a second agent without completing steps 2-5 first.

- **FR-010**: The Orchestrator SHALL NEVER pre-queue, batch, or parallelize agent invocations. Every delegation decision is made fresh from current state.

---

### 4.5 Error Recovery

- **FR-011**: When an agent invocation fails (agent reports error, produces no output, or times out), the Orchestrator SHALL:
  1. Record the failure in `error_log` in `.sdd/state.md`
  2. Increment `retry_count`
  3. If `retry_count` < 2 (max retries), retry the same agent with the same input
  4. If `retry_count` >= 2, escalate to the user with: the error summary, the agent name, the WP (if applicable), and the full error log

- **FR-012**: When the same WP fails review 3 times (3 review cycles returning lane=to_do), the Orchestrator SHALL halt and escalate to the user with:
  1. All review feedback from the 3 cycles
  2. The WP file path
  3. A summary of what was attempted

- **FR-013**: After a successful agent invocation, the Orchestrator SHALL reset `retry_count` to 0.

#### Implementation Contract -- Error Recovery

**Inputs**: Agent invocation result (success/failure), current retry_count.
**Outputs**: Updated state file, retry invocation or escalation message.
**Error behaviors**: retry_count >= 2 - escalate. Review fail 3x - halt. Write failure - halt.

---

### 4.6 Universal Escalation Support

- **FR-014**: The Orchestrator SHALL support escalation from any agent. When a delegated agent reports an escalation (e.g., spec ambiguity, environment issue, unresolvable conflict), the Orchestrator SHALL:
  1. Record the escalation in the state file (`last_result: escalated`)
  2. Present the escalation to the user with full context
  3. Wait for user response before continuing

- **FR-015**: When the user resolves an escalation, the Orchestrator SHALL determine which agent to re-invoke based on the resolution (not necessarily the same agent that escalated).

---

### 4.7 Status Reporting

- **FR-016**: After every agent completion, the Orchestrator SHALL display a status report:

  ```
  ## Pipeline Status

  **Last action**: {agent} completed {what it did}
  **Result**: {success/needs-fixes/blocked/escalated}
  **Retries**: {retry_count}/2

  | Stage       | Status           |
  |-------------|------------------|
  | Ideation    | {done/pending}   |
  | Spec        | {done/pending}   |
  | Planning    | {done/pending}   |
  | WP{NN} Impl | {lane value}     |
  | WP{NN} Review | {passed/failed/pending} |
  | WP{NN} Docs | {done/pending}   |

  **Next action**: {description of next action}
  ```

- **FR-017**: The Orchestrator SHALL use `manage_todo_list` to maintain a high-level pipeline tracker visible to the user throughout the session.

---

## 5. User Stories

### US-01 -- Cross-Session Pipeline Resumption (Priority: P1)

**As a** Human Developer, **I want** the pipeline state to persist across VS Code sessions, **so that** I can close VS Code and resume where I left off.

**Why P1**: Long pipelines span multiple sessions. Without persistence, the Orchestrator starts from scratch.

**Independent Test**: Start pipeline, complete 2 WPs, close VS Code, reopen, invoke Orchestrator. Verify: it starts at WP03 (not WP01).

**Acceptance Scenarios**:
1. **Given** WP01 and WP02 are lane=done and `.sdd/state.md` says `current_wp: WP03, pipeline_stage: implementation`, **When** the Orchestrator starts, **Then** it verifies state against WP frontmatter and proceeds to implement WP03.
2. **Given** `.sdd/state.md` says `current_wp: WP03, pipeline_stage: review` but WP03's lane is `done`, **When** the Orchestrator starts, **Then** it updates state.md to reflect the actual state and proceeds to documentation for WP03 (or next WP if docs already done).

---

### US-02 -- Automatic Retry on Transient Failure (Priority: P1)

**As a** Human Developer, **I want** failed agents automatically retried before I am interrupted, **so that** transient failures do not require manual intervention.

**Why P1**: Transient failures (context overflow, tool timeout) are common. Auto-retry reduces friction.

**Independent Test**: Simulate agent failure on first attempt, success on second. Verify: pipeline continues without user intervention.

**Acceptance Scenarios**:
1. **Given** the Coder fails on WP03 with a transient error, **When** retry_count is 0, **Then** the Orchestrator retries the Coder for WP03 and records the failure in error_log.
2. **Given** the Coder fails on WP03 twice (retry_count = 2), **When** the second retry fails, **Then** the Orchestrator escalates to the user with the error log.

---

### US-03 -- Docs Agent Integration (Priority: P2)

**As a** Human Developer, **I want** the Orchestrator to trigger the Docs Agent after WP approval, **so that** documentation is always current.

**Why P2**: Documentation generation is important. The Orchestrator must know about the Docs Agent in the pipeline.

**Independent Test**: Approve WP03. Verify: Orchestrator invokes Docs Agent before starting WP04.

**Acceptance Scenarios**:
1. **Given** WP03 review passes (lane=done), **When** Orchestrator reads updated state, **Then** it invokes the Docs Agent for WP03 before advancing to WP04.
2. **Given** WP03 review fails (lane=to_do), **When** Orchestrator reads updated state, **Then** it invokes the Coder (not Docs Agent) to fix WP03.

---

### Edge Cases

- What happens when `.sdd/state.md` is corrupted or has invalid YAML? Orchestrator recreates it from WP frontmatter ground truth and logs a warning.
- What happens when a WP has no dependencies listed? It is always eligible for implementation.
- What happens when the Docs Agent fails? Treated like any other agent failure: retry up to 2 times, then escalate.

---

## 6. User Flows

### 6.1 Full Pipeline Run (Happy Path)

1. User invokes Orchestrator with "run full cycle."
2. Orchestrator reads `.sdd/state.md` (creates if missing).
3. Orchestrator cross-verifies state against WP frontmatter.
4. Orchestrator updates todo list with pipeline stages.
5. Orchestrator determines next action from Decision Table.
6. Orchestrator invokes the agent selected by the Decision Table with its prompt template.
7. Agent completes. Orchestrator re-reads state.
8. Orchestrator updates `.sdd/state.md` and todo list.
9. Orchestrator displays status report.
10. Repeat steps 5-9 until all WPs are lane=done and documented.
11. Orchestrator reports pipeline complete.

### 6.2 Cross-Session Resume

1. User opens VS Code and invokes Orchestrator.
2. Orchestrator reads `.sdd/state.md`.
3. State says `current_wp: WP03, pipeline_stage: implementation`.
4. Orchestrator reads WP03 frontmatter: lane=planned.
5. States are consistent. Orchestrator invokes Coder for WP03.

### 6.3 Recovery After Failure

1. Coder fails on WP03.
2. Orchestrator logs failure in error_log, increments retry_count to 1.
3. Orchestrator retries Coder for WP03.
4. Coder succeeds. Orchestrator resets retry_count to 0.
5. Orchestrator proceeds normally.

### 6.4 Escalation Flow

1. Coder fails on WP03 twice (retry_count = 2).
2. Orchestrator presents error log to user.
3. User provides guidance ("skip this WP" or "fix X and retry").
4. Orchestrator acts on user's decision.

---

## 7. Data Model

### 7.0 State Transitions for `pipeline_stage`

The `pipeline_stage` field SHALL follow these valid transitions:

| From | To | Trigger |
|------|----|---------|
| idle | ideation | User provides an idea description |
| idle | specification | Existing brief found without a spec |
| idle | planning | Existing spec found without WPs |
| idle | implementation | Existing WPs found with lane=planned |
| ideation | specification | Ideation agent completes brief |
| specification | planning | Spec status set to Validated |
| planning | implementation | Planner produces WP files |
| implementation | review | Coder sets WP lane=for_review |
| review | documentation | Review Coordinator sets WP lane=done |
| review | implementation | Review Coordinator sets WP lane=to_do |
| documentation | implementation | Docs Agent completes and next WP exists |
| documentation | complete | Docs Agent completes and no WPs remain |
| implementation | complete | All WPs lane=done and documented |

Any transition not in this table is invalid. The Orchestrator SHALL NOT set `pipeline_stage` to a value that is not reachable from the current value.

### 7.1 State File Entity

| Field | Type | Required | Default | Validation |
|-------|------|----------|---------|------------|
| pipeline_stage | string (enum) | yes | "idle" | One of: idle, ideation, specification, planning, implementation, review, documentation, complete |
| current_spec | string (path) or null | yes | null | Valid path or null |
| current_wp | string or null | yes | null | Format: WP followed by 2 digits (e.g., WP01) or null |
| last_agent | string or null | yes | null | Agent name or null |
| last_result | string (enum) or null | yes | null | One of: success, failed, escalated, null |
| retry_count | integer | yes | 0 | >= 0, max 2 before escalation |
| error_log | array of ErrorEntry | yes | [] | Max 50 entries (oldest pruned) |
| updated_at | string (ISO 8601) | yes | creation time | Valid ISO 8601 |

### 7.2 ErrorEntry

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| agent | string | yes | Agent name |
| wp | string or null | yes | WP identifier or null |
| error_summary | string | yes | 1-500 characters |
| timestamp | string (ISO 8601) | yes | Valid ISO 8601 |

---

## 8. API / Interface Design

### 8.1 Orchestrator Invocation

**Invocation**: User sends a message to the Orchestrator agent (e.g., "run full cycle", "implement all WPs", "continue pipeline").

**Agent prompts for delegation**:

| Target Agent | Prompt Template |
|-------------|----------------|
| Ideation | "Create an ideation brief for: {user's description}" |
| Spec Architect | "Develop the brainstorming session output into a full specification. The brief is at {brief_path}" |
| Planner | "Decompose the specification into work packages. The spec is at {spec_path}" |
| Coder | "Implement {wp_id} - {wp_title}. The plan is at {wp_path}" |
| Review Coordinator | "Review {wp_id}. It is at lane=for_review. The plan is at {wp_path}" |
| Docs Agent | "{wp_id} has been approved. WP file: {wp_path}. Spec: {spec_path}. Update documentation." |

---

## 9. Architecture

### 9.1 System Design

The Orchestrator is a state machine. It reads state, makes a decision, delegates to one agent, reads the updated state, and loops. It never performs the work itself.

```
[Read State] -> [Verify State vs Frontmatter] -> [Decide] -> [Delegate ONE Agent] -> [Read Updated State] -> [Update State File] -> [Report] -> [Loop]
```

### 9.2 Key Design Decisions

**Decision 1: State file + WP frontmatter dual verification**
- **Rationale**: Redundancy catches inconsistencies. State file enables cross-session continuity. WP frontmatter is ground truth because specialist agents modify it.
- **Alternatives**: State file only (stale risk), WP frontmatter only (no cross-session awareness).
- **Consequences**: Small overhead per cycle to verify, but prevents state drift.

**Decision 2: Strict sequential execution (fix pre-queuing bug)**
- **Rationale**: Pre-queuing caused the Orchestrator to invoke agents based on stale state. The fix is architectural: never invoke based on state you have not just read.
- **Alternatives**: Batch invocations with rollback, parallel with locking.
- **Consequences**: Slower (sequential), but correct. Avoids race conditions and stale-state decisions.

**Decision 3: Retry before escalate**
- **Rationale**: Transient failures are common with LLM-based agents. Retrying once or twice avoids unnecessary human interruption.
- **Alternatives**: No retry (always escalate), unlimited retry.
- **Consequences**: 2 retries max keeps the pipeline moving without masking persistent failures.

**Decision 4: Docs Agent in the per-WP loop**
- **Rationale**: User chose incremental doc generation after each WP approval.
- **Alternatives**: Batch docs after all WPs, on-demand only.
- **Consequences**: More agent invocations, but docs always stay current.

---

## 10. Non-Functional Requirements

### 10.1 Performance
- State file read/write SHALL complete in under 1 second.
- Decision logic (state assessment + next action) SHALL complete in under 5 seconds.

### 10.2 Security
- The state file contains no secrets or credentials.
- Error logs SHALL NOT contain full stack traces with sensitive paths; only summaries.

---

## 11. Test Requirements

### 11.1 Test Strategy

All requirements SHALL be verified through BDD acceptance scenarios. Since the Orchestrator is an agent mode prompt (not compiled code), tests are defined as behavioral scenarios verified by manual walkthrough or integration testing against the agent framework.

### 11.2 BDD / Acceptance Tests

```gherkin
Feature: Orchestrator V2 - State Tracking

  Scenario: Cross-session resume
    Given .sdd/state.md says current_wp=WP03 and pipeline_stage=implementation
    And WP03 frontmatter has lane=planned
    When the Orchestrator starts
    Then it verifies state against frontmatter
    And proceeds to implement WP03

  Scenario: State file vs frontmatter discrepancy
    Given .sdd/state.md says current_wp=WP03 and pipeline_stage=review
    And WP03 frontmatter has lane=done
    When the Orchestrator starts
    Then it updates state.md to reflect lane=done
    And proceeds to documentation for WP03

Feature: Orchestrator V2 - Error Recovery

  Scenario: Automatic retry on transient failure
    Given the Coder agent fails on WP03
    And retry_count is 0
    When the Orchestrator handles the failure
    Then it logs the error in error_log
    And increments retry_count to 1
    And retries the Coder for WP03

  Scenario: Escalation after max retries
    Given the Coder agent fails on WP03
    And retry_count is 2
    When the Orchestrator handles the failure
    Then it escalates to the user with the full error log
    And does not retry

Feature: Orchestrator V2 - Docs Agent Integration

  Scenario: Docs Agent after WP approval
    Given WP03 review passes and lane is set to done
    When the Orchestrator reads updated state
    Then it invokes the Docs Agent for WP03
    And does not start WP04 until docs complete

  Scenario: No Docs Agent on review failure
    Given WP03 review fails and lane is set to to_do
    When the Orchestrator reads updated state
    Then it invokes the Coder to fix WP03
    And does not invoke the Docs Agent

Feature: Orchestrator V2 - Sequential Execution

  Scenario: No pre-queuing
    Given the Coder completes WP03 successfully
    When the Orchestrator processes the result
    Then it reads updated state from .sdd/ before deciding the next action
    And it does not invoke Review Coordinator until state is verified
```

---

## 12. Constraints & Assumptions

### Constraints
- Operates within VS Code Copilot Chat agent framework.
- Cannot invoke agents in parallel (single-threaded subagent invocation).
- State file is markdown with YAML frontmatter (no database).

### Assumptions
1. All pipeline agents (Ideation, Spec Architect, Planner, Coder, Review Coordinator, Docs Agent) are implemented and available.
2. WP files use `lane` in YAML frontmatter with values: planned, doing, for_review, to_do, done.
3. The `runSubagent` tool returns control to the Orchestrator after each agent completes.

---

## 13. Out of Scope

- **Agent parallelism**: Sequential only. No concurrent WP processing.
- **Pipeline analytics**: Tracking success rates and skill performance is future work (P3).
- **Multi-pipeline support**: Only one pipeline runs at a time per workspace.
- **Automatic rollback**: On persistent failure, the Orchestrator escalates rather than rolling back.

---

## 14. Open Questions

None remaining.

---

## 15. Glossary

- **Pre-queuing bug**: A defect where the Orchestrator decides the next agent before reading the result of the current agent, leading to decisions based on stale state.
- **State drift**: When the state file and WP frontmatter disagree, typically because the state file was not updated after an agent modified a WP.
- **Escalation**: Pausing the pipeline and presenting a decision or error to the human developer.

---

## 16. Traceability Matrix

| FR ID | Requirement Summary | User Story | Acceptance Scenario | Test Type | Test Section Ref |
|-------|-------------------|------------|--------------------|-----------|----|
| FR-001 | State file with YAML frontmatter | US-01 | US-01 Scenario 1 | BDD | 11.2 |
| FR-002 | Create state file if missing | US-01 | US-01 Scenario 1 | BDD | 11.2 |
| FR-003 | Update state file after invocation | US-01 | US-01 Scenario 1 | BDD | 11.2 |
| FR-004 | State vs frontmatter verification | US-01 | US-01 Scenario 1, 2 | BDD | 11.2 |
| FR-005 | Orchestrator does not modify WP frontmatter | US-01 | US-01 Scenario 1, 2 | BDD | 11.2 |
| FR-006 | Updated pipeline sequence with Docs Agent | US-03 | US-03 Scenario 1 | BDD | 11.2 |
| FR-007 | Docs Agent after WP approval | US-03 | US-03 Scenario 1 | BDD | 11.2 |
| FR-008 | No Docs on unapproved WPs | US-03 | US-03 Scenario 2 | BDD | 11.2 |
| FR-009 | Strict sequential execution | US-01, US-02 | Sequential Scenario | BDD | 11.2 |
| FR-010 | No pre-queuing or parallelization | US-01 | Sequential Scenario | BDD | 11.2 |
| FR-011 | Error recovery with retry | US-02 | US-02 Scenario 1, 2 | BDD | 11.2 |
| FR-012 | Review fail 3x halt | US-02 | US-02 Edge case | BDD | 11.2 |
| FR-013 | Reset retry_count on success | US-02 | US-02 Scenario 1 | BDD | 11.2 |
| FR-014 | Universal escalation support | US-02 | US-02 Edge case | BDD | 11.2 |
| FR-015 | Re-invoke after escalation resolution | US-02 | US-02 Edge case | BDD | 11.2 |
| FR-016 | Status report after every agent | US-01, US-03 | US-01 Scenario 1 | BDD | 11.2 |
| FR-017 | Todo list pipeline tracker | US-01 | US-01 Scenario 1 | BDD | 11.2 |

---

## 17. Technical References

- VS Code Copilot Chat Agent Framework, https://code.visualstudio.com/docs/copilot/chat/chat-agent-mode, consulted 2026-04-05

---

## 18. Version History

| Version | Date | Author | Summary of Changes |
|---------|------|--------|--------------------|
| 1.0 | 2026-04-05 | Spec Architect | Initial specification |
| 1.1 | 2026-04-05 | Spec Architect | Validation: added error behavior to FR-002/FR-003, state transitions table, completed traceability matrix (all 17 FRs), fixed ambiguous language, added section 11.1, generated companion artifacts |
