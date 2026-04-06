---
lane: planned
---

# WP43 - Error-Handling Policy & Acceptance Criteria RACI

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/010-sdd-pipeline-hardening.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP40 |
| Goal | Document the error-handling policy and the maker/checker acceptance criteria pattern so pipeline behavior is explicit and understandable |
| Status | Not Started |
| Independent Test | Read `.sdd/docs/architecture.md` and find the "Design Decision: Error-Handling Policy" section categorizing every agent. Read any agent file and find the error policy reference comment. Read the Coder agent and find "Responsible (maker)" label. |
| Parallelisable | Yes (with WP41, WP42) |
| Prompt | `.sdd/plans/WP43-error-policy-raci.md` |

## Objective

Document two key pipeline conventions that are currently implicit: (1) the error-handling asymmetry between critical-path agents (HALT on failure) and advisory agents (best-effort), and (2) the maker/checker pattern for acceptance criteria checkboxes where the Coder checks boxes and the Reviewer verifies them. Both conventions are documented in central locations (architecture.md, developer-guide.md) and referenced from agent instruction files.

## Spec References

FR-028, FR-029, FR-030, FR-031, FR-032, FR-033, Section 4.5 (Error-Handling Policy), Section 4.6 (Acceptance Criteria RACI), Section 9.4 (Design Decision 1), US-03, US-09

## Tasks

### T43-01 - Add error-handling policy to architecture.md

- **Description**: Add a subsection titled "Design Decision: Error-Handling Policy" within the Design Decisions section of `.sdd/docs/architecture.md`. Document the critical-path vs advisory categorization for every pipeline agent.
- **Spec refs**: FR-028, FR-029, Section 4.5
- **Parallel**: No (foundation for T43-02)
- **Acceptance criteria**:
  - [ ] `.sdd/docs/architecture.md` contains a subsection titled "Design Decision: Error-Handling Policy" (FR-028)
  - [ ] Critical-path agents listed with HALT behavior: Spec Architect, Planner, Coder (FR-029)
  - [ ] Advisory agents listed with best-effort behavior: Review Coordinator, Docs Agent (FR-029)
  - [ ] Orchestrator, Ideation, and Brainstorming agents are also categorized (FR-029)
  - [ ] Rationale is provided for each categorization
  - [ ] Given a user reads architecture.md, they find error-handling guidance with every agent categorized (US-03 Scenario 1)
- **Test requirements**: content (grep search), BDD (US-03 Scenario 1)
- **Depends on**: none
- **Implementation Guidance**:
  - If architecture.md does not have a Design Decisions section, create it
  - Critical-path rationale: errors compound downstream, so halting prevents cascading failures
  - Advisory rationale: partial output is still valuable, continuation provides more value than halting
  - Files to modify: `.sdd/docs/architecture.md`

### T43-02 - Add error policy reference to all agent files

- **Description**: Add a one-line reference comment to each of the 8 agent files: `<!-- Error policy: See .sdd/docs/architecture.md, Design Decision: Error-Handling Policy -->`.
- **Spec refs**: FR-030, Section 4.5
- **Parallel**: Yes (each file can be edited independently)
- **Acceptance criteria**:
  - [ ] All 8 agent files in `.github/agents/` contain the error policy reference comment (FR-030)
  - [ ] The comment text is exactly: `<!-- Error policy: See .sdd/docs/architecture.md, Design Decision: Error-Handling Policy -->`
  - [ ] Given a user reads any agent file, they find a reference comment pointing to the architecture doc (US-03 Scenario 2)
- **Test requirements**: content (grep search across all agent files), BDD (US-03 Scenario 2)
- **Depends on**: T43-01
- **Implementation Guidance**:
  - The 8 agent files: orchestrator, coder, review-coordinator, docs-agent, planner, spec-architect, ideation, brainstorming
  - Place the comment near the top of each file, after YAML frontmatter
  - Files to modify: all `.github/agents/*.agent.md` files

### T43-03 - Add Responsible (maker) label to Coder agent

- **Description**: Update the Coder agent instructions to explicitly label the Coder's acceptance criteria checkbox role as "Responsible (maker)".
- **Spec refs**: FR-031, Section 4.6
- **Parallel**: Yes (with T43-04)
- **Acceptance criteria**:
  - [ ] Coder agent instructions label acceptance criteria handling as "Responsible (maker)" (FR-031)
  - [ ] The existing checkbox behavior is preserved -- only the label is added
  - [ ] Given a maintainer reads the Coder agent, they find the role labeled "Responsible (maker)" (US-09 Scenario 1)
- **Test requirements**: content (grep search), BDD (US-09 Scenario 1)
- **Depends on**: none
- **Implementation Guidance**:
  - Find the acceptance criteria handling section in coder.agent.md
  - Add the explicit "Responsible (maker)" label without changing existing behavior
  - If no explicit acceptance criteria section exists, add one
  - Files to modify: `.github/agents/coder.agent.md`

### T43-04 - Add Accountable/Verifier (checker) label to Review Coordinator

- **Description**: Update the Review Coordinator agent instructions to explicitly label its acceptance criteria verification role as "Accountable/Verifier (checker)".
- **Spec refs**: FR-032, Section 4.6
- **Parallel**: Yes (with T43-03)
- **Acceptance criteria**:
  - [ ] Review Coordinator instructions label acceptance criteria handling as "Accountable/Verifier (checker)" (FR-032)
  - [ ] The existing verification behavior is preserved -- only the label is added
  - [ ] The role description clarifies this is intentional dual-touch, not redundancy
- **Test requirements**: content (grep search)
- **Depends on**: none
- **Implementation Guidance**:
  - Find the acceptance criteria verification section in review-coordinator.agent.md
  - Add the explicit "Accountable/Verifier (checker)" label
  - Files to modify: `.github/agents/review-coordinator.agent.md`

### T43-05 - Document maker/checker pattern in developer guide

- **Description**: Add an "Acceptance Criteria Ownership" section to `.sdd/docs/developer-guide.md` documenting the maker/checker pattern: the Coder checks boxes, the Reviewer verifies them, and this is intentional dual-touch.
- **Spec refs**: FR-033, Section 4.6
- **Parallel**: No
- **Acceptance criteria**:
  - [ ] Developer guide contains an "Acceptance Criteria Ownership" section (FR-033)
  - [ ] The section documents: Coder = Responsible (maker), Reviewer = Accountable/Verifier (checker)
  - [ ] The section explains this is intentional dual-touch, not redundancy
  - [ ] Given a maintainer reads the developer guide, they find the maker/checker pattern explanation (US-09 Scenario 2)
- **Test requirements**: content (grep search), BDD (US-09 Scenario 2)
- **Depends on**: none
- **Implementation Guidance**:
  - If developer-guide.md does not exist, create the relevant section in a new file
  - Include the RACI assignments and rationale
  - Files to modify: `.sdd/docs/developer-guide.md`

### T43-06 - Verify cross-file RACI consistency

- **Description**: Verify that the RACI labels in the Coder agent, Review Coordinator agent, and developer guide are consistent with each other.
- **Spec refs**: FR-031, FR-032, FR-033, Section 11.3
- **Parallel**: No (verification task)
- **Acceptance criteria**:
  - [ ] RACI descriptions are consistent across coder.agent.md, review-coordinator.agent.md, and developer-guide.md
  - [ ] Coder is labeled "Responsible (maker)" in all locations
  - [ ] Reviewer is labeled "Accountable/Verifier (checker)" in all locations
- **Test requirements**: integration (cross-file comparison)
- **Depends on**: T43-03, T43-04, T43-05
- **Implementation Guidance**:
  - Cross-reference Section 11.3 integration test: "RACI labels -> Developer guide"
  - Ensure terminology matches exactly across all three files

## Implementation Notes

- All deliverables are documentation updates (markdown files) -- no executable code
- The error-handling categorization addresses a key user confusion point: why some failures halt and others continue
- The RACI documentation addresses a maintenance risk: someone removing the "duplicate" checkbox handling
- Agent file reference comments (T43-02) are non-functional -- they are human-readable documentation only
- These changes do not modify agent behavior, only document existing behavior explicitly

## Risks & Mitigations

- **Risk**: Architecture.md may not have a Design Decisions section yet. **Mitigation**: Create the section if absent.
- **Risk**: Developer guide may not exist. **Mitigation**: Create the file if absent.

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
