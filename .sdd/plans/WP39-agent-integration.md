---
lane: planned
---

# WP39 - Agent Integration

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/009-research-skill-ideation.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | WP38 |
| Goal | Integrate the Research Skill into the Ideation and Brainstorming agents so both dispatch structured research and produce enriched briefs with cited sources |
| Status | Not Started |
| Independent Test | Describe an idea to the Ideation Agent. Verify: Research Skill is dispatched, brief contains Research Findings section with at least 2 cited sources, Risk Assessment section, and Technical Feasibility section |
| Parallelisable | No (single WP modifying 2 agent files sequentially) |
| Prompt | `.sdd/plans/WP39-agent-integration.md` |

## Objective

Modify the Ideation Agent (`.github/agents/ideation.agent.md`) and Brainstorming Agent (`.github/agents/brainstorming.agent.md`) to dispatch the Research Skill via `runSubagent` instead of performing ad-hoc inline research. Both agents' brief output templates are enriched with three new sections: Research Findings, Risk Assessment, and Technical Feasibility. All claims about external technologies or competitors must include source citations. This ensures research-backed briefs with traceable evidence flow into the Spec Architect.

## Spec References

FR-008, FR-009, FR-010, FR-011, FR-012, FR-013, FR-014, Section 5 (US-01, US-02), Section 6 (User Flows), Section 9.1 (Directory Structure -- MODIFIED agents), Section 11.2 (BDD Scenarios)

## Tasks

### T39-01 - Add Research Skill dispatch to Ideation Agent workflow

- **Description**: Modify the Ideation Agent's Discovery phase to dispatch the Research Skill via `runSubagent` with scope `[web, codebase]` after the user describes their idea, before asking clarifying questions. Add error handling for dispatch failures.
- **Spec refs**: FR-008, Section 6.1 (User Flow steps 2-6)
- **Parallel**: No (foundational change to ideation workflow)
- **Acceptance criteria**:
  - [ ] The Ideation Agent SHALL dispatch the Research Skill with scope `[web, codebase]` after the user describes their idea, before asking clarifying questions (FR-008)
  - [ ] The dispatch SHALL use the prompt template from Section 8.1 with topic derived from the user's idea description
  - [ ] If the Research Skill dispatch fails, the Ideation Agent SHALL log the failure and proceed without research, noting "Research unavailable" in the brief (FR-008 Error)
  - [ ] Given a user describes "build a CLI tool for database migrations," when the Ideation Agent processes the idea, then it dispatches the Research Skill (US-01 Scenario 1)
- **Test requirements**: BDD (Feature: Ideation with Research)
- **Depends on**: none (within this WP)
- **Implementation Guidance**:
  - Modify the Discovery phase (section 1) of `ideation.agent.md`
  - Replace the current "Mandatory competitive research" block that uses direct `fetch_webpage` calls
  - The dispatch prompt should follow Section 8.1:
    ```
    Research the following topic and write findings to {output_file}.
    Topic: {topic}
    Scope: web, codebase
    Questions:
    1. What existing tools/products solve this problem?
    2. What patterns and conventions exist in the codebase?
    ```
  - The output_file should be a temporary path (e.g., `.sdd/research-{timestamp}.md`)
  - After dispatch returns, the agent reads the output file to inform its clarifying questions
  - Error handling: wrap dispatch in a try/log pattern; on failure, set a flag "research_unavailable = true"
  - Preserve the existing `web_research_policy` section but reference the Research Skill as the primary mechanism
  - Files to modify: `.github/agents/ideation.agent.md`

### T39-02 - Add enriched brief format sections to Ideation Agent

- **Description**: Update the Ideation Agent's brief template to include three new sections: Research Findings, Risk Assessment, and Technical Feasibility. The Research Findings section is sourced from the Research Skill's output file.
- **Spec refs**: FR-009, FR-010, Section 7.2 (Research Output Sections)
- **Parallel**: No (depends on T39-01)
- **Acceptance criteria**:
  - [ ] The Ideation Agent SHALL include a "Research Findings" section in the brief output, sourced from the Research Skill's output file (FR-009)
  - [ ] If the research output file is empty or missing, the section SHALL state "No research findings available" with the reason (FR-009 Error)
  - [ ] The brief output format SHALL include a "Research Findings" section covering competitive landscape, analogous solutions, technology feasibility (FR-010.1)
  - [ ] The brief output format SHALL include a "Risk Assessment" section with risks identified from research with likelihood and impact (FR-010.2)
  - [ ] The brief output format SHALL include a "Technical Feasibility" section with confirmed feasible vs needs validation, with evidence from research (FR-010.3)
  - [ ] Given the user describes an idea, when the Ideation Agent completes the brief, then the brief contains a Research Findings section (BDD: Brief includes research findings)
- **Test requirements**: BDD (Scenario: Brief includes research findings)
- **Depends on**: T39-01
- **Implementation Guidance**:
  - Add three new sections to the `<brief_template>` in `ideation.agent.md`:
    ```markdown
    ## Research Findings
    Competitive landscape, analogous solutions, and technology feasibility sourced from research.
    Cite all sources with URLs.

    ## Risk Assessment
    | Risk | Likelihood | Impact | Source |
    |------|-----------|--------|--------|

    ## Technical Feasibility
    | Item | Status | Evidence | Source |
    |------|--------|----------|--------|
    ```
  - Status values for Technical Feasibility: "Confirmed feasible" or "Needs validation" (matching TechnicalFeasibility.status from data model)
  - Risk Assessment likelihood/impact values: "low", "medium", "high" (matching RiskAssessment from data model)
  - When research is unavailable: Research Findings says "No research findings available"; Risk Assessment says "Not assessed"; Technical Feasibility says "Not assessed"
  - Files to modify: `.github/agents/ideation.agent.md`

### T39-03 - Add source citation requirements to Ideation Agent

- **Description**: Add citation requirements to the Ideation Agent so every claim about an external technology, competitor, or pattern includes a source URL. Unverified claims receive an "[Unverified]" prefix.
- **Spec refs**: FR-011
- **Parallel**: Yes (independent of T39-04)
- **Acceptance criteria**:
  - [ ] The Ideation Agent SHALL cite sources in the brief (FR-011)
  - [ ] Every claim about an external technology, competitor, or pattern SHALL have a source URL (FR-011)
  - [ ] If no sources were found for a claim, the claim SHALL be prefixed with "[Unverified]" and omit the source citation (FR-011 Error)
  - [ ] Given the Research Skill finds 3 competing tools, when the brief is written, then each competitor has a source URL (US-01 Scenario 2)
  - [ ] The brief SHALL contain at least 2 cited sources (BDD: Brief includes research findings -- "at least 2 sources are cited")
- **Test requirements**: BDD (Scenario: Brief includes research findings)
- **Depends on**: T39-02
- **Implementation Guidance**:
  - Add a citation rule to the `<rules>` section of `ideation.agent.md`:
    ```
    - ALWAYS cite sources for claims about external technologies, competitors, or patterns -- include the source URL
    - If no source is available for a claim, prefix it with "[Unverified]"
    ```
  - Also add citation instructions to the brief template sections (Research Findings, Competitive Landscape)
  - Citation format: `[Title](URL), consulted YYYY-MM-DD`
  - Known pitfall: the agent may make claims based on its training data without a URL; the "[Unverified]" prefix handles this
  - Files to modify: `.github/agents/ideation.agent.md`

### T39-04 - Add Research Skill dispatch to Brainstorming Agent workflow

- **Description**: Modify the Brainstorming Agent to dispatch the Research Skill via `runSubagent` with scope `[web, codebase, packages]` at specific trigger points: exploring technology alternatives, evaluating competing approaches, and validating assumptions about external systems.
- **Spec refs**: FR-012, Section 6.2 (User Flow steps 1-4)
- **Parallel**: Yes (independent of T39-03)
- **Acceptance criteria**:
  - [ ] The Brainstorming Agent SHALL dispatch the Research Skill with scope `[web, codebase, packages]` when exploring technology alternatives (FR-012.1)
  - [ ] The Brainstorming Agent SHALL dispatch the Research Skill when evaluating competing approaches (FR-012.2)
  - [ ] The Brainstorming Agent SHALL dispatch the Research Skill when validating assumptions about external systems (FR-012.3)
  - [ ] If the Research Skill dispatch fails, the Brainstorming Agent SHALL log the failure and continue without research-backed data, noting the limitation to the user (FR-012 Error)
- **Test requirements**: BDD (Feature: Brainstorming with Research)
- **Depends on**: none (within this WP)
- **Implementation Guidance**:
  - The brainstorming agent has different trigger points from the ideation agent -- it dispatches research on-demand during the session, not at the start
  - Add dispatch instructions to the `<web_research_policy>` section and to the exploration technique sections
  - The scope includes "packages" (unlike ideation's [web, codebase]) because brainstorming sessions evaluate specific packages
  - Trigger points map to existing workflow phases: "Divergent techniques" and "Convergent techniques"
  - The dispatch prompt should adapt the topic and questions based on the current brainstorming context
  - Error handling: on dispatch failure, log and continue with the user's own knowledge; note "Research unavailable for this comparison" to the user
  - Files to modify: `.github/agents/brainstorming.agent.md`

### T39-05 - Add research-backed pros/cons to Brainstorming Agent

- **Description**: Modify the Brainstorming Agent so that when presenting alternatives, it includes research-backed pros/cons from the Research Skill's findings.
- **Spec refs**: FR-013
- **Parallel**: No (depends on T39-04)
- **Acceptance criteria**:
  - [ ] Research findings SHALL inform the brainstorming Q&A (FR-013)
  - [ ] When presenting alternatives, the agent SHALL include research-backed pros/cons (FR-013)
  - [ ] If no alternatives were found by research, the agent SHALL present user-provided alternatives and note "No research data available for comparison" (FR-013 Error)
  - [ ] Given a brainstorming session reaches a technology decision, when the agent dispatches the Research Skill, then alternatives are presented with version numbers and status, and sources are cited (BDD: Technology comparison backed by research)
- **Test requirements**: BDD (Scenario: Technology comparison backed by research)
- **Depends on**: T39-04
- **Implementation Guidance**:
  - Modify the "Convergent techniques" section to reference research findings when building trade-off matrices
  - When the Research Skill returns a TechnologyEvaluation table, the agent should present it directly in the conversation
  - Format: include version numbers, maintenance status, license, and source URLs alongside each alternative
  - When research returns no results: present the user's own alternatives with a disclaimer
  - Add to the session_tracking rules: track "Research Findings" as a category in the todo list
  - Files to modify: `.github/agents/brainstorming.agent.md`

### T39-06 - Add enriched brief format to Brainstorming Agent

- **Description**: Update the Brainstorming Agent's brief template to include the same enriched format as the Ideation Agent: Research Findings, Risk Assessment, and Technical Feasibility sections, with appropriate defaults when no research was performed.
- **Spec refs**: FR-014
- **Parallel**: No (depends on T39-05)
- **Acceptance criteria**:
  - [ ] The Brainstorming Agent SHALL produce a brief with the same enriched format as the Ideation Agent (FR-014, same as FR-010)
  - [ ] If no research was performed during the session, the Research Findings section SHALL state "No research performed" (FR-014 Error)
  - [ ] If no research was performed during the session, the Risk Assessment section SHALL state "Not assessed" (FR-014 Error)
  - [ ] Source citations SHALL follow the same format as the Ideation Agent ([Title](URL), consulted date)
- **Test requirements**: BDD (Section 11.2)
- **Depends on**: T39-05
- **Implementation Guidance**:
  - Add the same three sections to the brainstorming agent's brief template as T39-02 added to ideation
  - The brainstorming brief template is at the bottom of `brainstorming.agent.md` (look for the `<brief_template>` section)
  - Default values differ from ideation: brainstorming uses "No research performed" (not "No research findings available") because brainstorming research is on-demand, not automatic
  - Ensure consistency with the data model artifact: ResearchFindings, RiskAssessment, TechnicalFeasibility types
  - Files to modify: `.github/agents/brainstorming.agent.md`

### T39-07 - Verify both agents dispatch Research Skill correctly

- **Description**: Perform integration verification by checking that both modified agents correctly reference the Research Skill and that the dispatch prompt matches the skill's expected input format.
- **Spec refs**: SC-001, SC-002, SC-003
- **Parallel**: No (final verification task)
- **Acceptance criteria**:
  - [ ] Ideation and Brainstorming agents use the Research Skill for all research tasks; no raw `fetch_webpage` or `grep_search` calls for research outside the skill (SC-001)
  - [ ] Brief output includes competitive analysis, technology evaluation, and risk assessment sourced from research; every brief has a "Research Findings" section with cited sources (SC-002)
  - [ ] A third agent can invoke the Research Skill by dispatching it as a subagent with zero changes to the skill (SC-003)
  - [ ] The dispatch prompt used by both agents matches the template in Section 8.1
  - [ ] Both agents' enriched brief sections match the EnrichedBriefSections structure from the data model artifact
- **Test requirements**: BDD (all scenarios from Section 11.2)
- **Depends on**: T39-03, T39-06
- **Implementation Guidance**:
  - Cross-check the dispatch prompt in both agents against Section 8.1
  - Verify the enriched brief sections in both agents' templates match exactly: same section names, same table columns, same default values
  - Search both agent files for any remaining direct `fetch_webpage` calls used for research purposes -- these should be replaced by Research Skill dispatch
  - Note: both agents may retain `fetch_webpage` for non-research purposes (e.g., fetching a specific URL the user provided); only research-pattern usage should go through the skill
  - Files to verify: `.github/agents/ideation.agent.md`, `.github/agents/brainstorming.agent.md`, `.github/skills/research/SKILL.md`

## Implementation Notes

- This WP modifies two existing agent files and does not create any new files
- Both agent files are extensive (~300+ lines each); modifications target specific sections (Discovery phase, web_research_policy, brief_template)
- The Ideation Agent dispatches research once at the start (scope: web, codebase); the Brainstorming Agent dispatches on-demand during the session (scope: web, codebase, packages)
- Both agents retain their existing `web_research_policy` sections but reference the Research Skill as the primary mechanism
- The enriched brief format must be identical across both agents (FR-014 explicitly references FR-010)
- "Testing" means manually invoking each agent, describing an idea, and verifying the output brief structure matches the BDD scenarios

## Parallel Opportunities

- T39-03 (ideation citations) and T39-04 (brainstorming dispatch) can be worked concurrently as they modify different agent files

## Risks & Mitigations

- **Risk**: Existing web_research_policy content in both agents may conflict with Research Skill dispatch instructions. **Mitigation**: Preserve the policy as a fallback reference but make the Research Skill the primary mechanism
- **Risk**: Brainstorming agent dispatches research on-demand multiple times per session, possibly exceeding the 5-minute timeout. **Mitigation**: Each dispatch is independent; the 5-minute limit applies per invocation, not per session
- **Risk**: Modifying the brief template may break existing brief structure expectations. **Mitigation**: New sections are additive -- they extend the template without removing existing sections

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
