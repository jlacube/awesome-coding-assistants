# Research Skill + Ideation/Brainstorming Improvements -- Specification

> **Source brief**: `.sdd/ideas/002-sdd-pipeline-v2-universal-skill-architecture.md`
> **Feature branch**: `009-research-skill-ideation`
> **Status**: Validated
> **Version**: 1.1

---

## 1. Overview

Create a shared Research Skill that performs web search, codebase analysis, and package registry lookups, then make it available to both the Ideation and Brainstorming agents. Additionally, deepen the brief output format of both agents so their output provides richer input for the Spec Architect V2. The Research Skill eliminates ad-hoc research patterns currently spread across agents and centralizes them into a reusable, well-structured skill.

---

## 2. Goals & Success Criteria

- **SC-001**: Ideation and Brainstorming agents use the Research Skill for all research tasks. Verified by: no raw `fetch_webpage` or `grep_search` calls for research outside the skill.
- **SC-002**: Brief output includes competitive analysis, technology evaluation, and risk assessment sourced from research. Verified by: every brief has a "Research Findings" section with cited sources.
- **SC-003**: Adding the Research Skill to a new agent requires zero code changes to the skill itself. Verified by: a third agent can invoke it by dispatching it as a subagent.

---

## 3. Users & Roles

- **Ideation Agent (invoker)**: Dispatches the Research Skill during idea exploration to gather competitive landscape, technology feasibility, and domain context.
- **Brainstorming Agent (invoker)**: Dispatches the Research Skill during deep brainstorming to explore alternatives, validate assumptions, and discover analogous solutions.
- **Human Developer (indirect user)**: Benefits from research-backed briefs with cited sources.

---

## 4. Functional Requirements

### 4.1 Research Skill

- **FR-001**: The Research Skill SHALL be implemented as `.github/skills/research/SKILL.md` and usable by any agent via subagent dispatch.

- **FR-002**: The Research Skill SHALL accept a research request specifying:
  - `topic`: The subject to research (string, 1-200 characters)
  - `scope`: One or more of: `web`, `codebase`, `packages` (array of strings)
  - `questions`: Specific questions to answer (array of strings, 1-10 items)
  - `output_file`: Path where findings SHALL be written (string)

- **FR-003**: For `web` scope, the skill SHALL:
  1. Search for the topic using `fetch_webpage` on relevant URLs
  2. Prioritize official documentation, GitHub repositories, and established architecture resources
  3. Extract: current status, latest version, known issues, community size, license
  4. Record source URL and date consulted for every finding

- **FR-004**: For `codebase` scope, the skill SHALL:
  1. Search the workspace using `grep_search` and `semantic_search`
  2. Identify existing patterns, conventions, frameworks, and configurations
  3. Surface existing code relevant to the research topic
  4. Note any technical constraints discovered

- **FR-005**: For `packages` scope, the skill SHALL:
  1. Look up packages on npm, PyPI, crates.io, or other registries via web fetch
  2. Check: latest version, maintenance activity (last publish date), download counts, license, known CVEs
  3. Compare alternatives when multiple packages serve the same purpose

- **FR-006**: The skill SHALL write findings to `output_file` in a structured format:

  ```markdown
  # Research Findings: {topic}

  ## Questions & Answers
  ### Q1: {question}
  **Answer**: {finding}
  **Sources**: [{title}]({url}), consulted {date}

  ## Competitive/Analogous Solutions
  | Solution | Approach | Strengths | Weaknesses |
  |----------|----------|-----------|------------|

  ## Technology Evaluation
  | Technology | Version | Status | License | Last Updated | Recommendation |
  |-----------|---------|--------|---------|-------------|---------------|

  ## Codebase Context
  - {relevant file}: {what it tells us}

  ## Risks & Concerns
  - {risk identified from research}
  ```

- **FR-007**: The skill SHALL complete research within 5 minutes. If a web fetch times out, skip that source and note it as "unavailable."

#### Implementation Contract -- Research Skill

**Inputs**: topic (string), scope (array of "web"|"codebase"|"packages"), questions (array of strings), output_file (string path).
**Outputs**: Structured markdown file at output_file.
**Error behaviors**: Web fetch timeout - skip source, note as unavailable. No results found - write "No findings" with explanation. Invalid scope value - ignore, log warning.

---

### 4.2 Ideation Agent Improvements

- **FR-008**: The Ideation Agent SHALL dispatch the Research Skill with scope `[web, codebase]` after the user describes their idea, before asking clarifying questions.
  - **Error**: If the Research Skill dispatch fails, the Ideation Agent SHALL log the failure and proceed without research, noting "Research unavailable" in the brief.

- **FR-009**: The Ideation Agent SHALL include a "Research Findings" section in the brief output, sourced from the Research Skill's output file.
  - **Error**: If the research output file is empty or missing, the section SHALL state "No research findings available" with the reason.

- **FR-010**: The brief output format SHALL include these additional sections (beyond current format):
  1. **Research Findings**: Competitive landscape, analogous solutions, technology feasibility
  2. **Risk Assessment**: Risks identified from research with likelihood and impact
  3. **Technical Feasibility**: Confirmed feasible vs needs validation, with evidence from research

- **FR-011**: The Ideation Agent SHALL cite sources in the brief. Every claim about an external technology, competitor, or pattern SHALL have a source URL.
  - **Error**: If no sources were found for a claim, the claim SHALL be prefixed with "[Unverified]" and omit the source citation.

---

### 4.3 Brainstorming Agent Improvements

- **FR-012**: The Brainstorming Agent SHALL dispatch the Research Skill with scope `[web, codebase, packages]` when:
  1. Exploring technology alternatives
  2. Evaluating competing approaches
  3. Validating assumptions about external systems
  - **Error**: If the Research Skill dispatch fails, the Brainstorming Agent SHALL log the failure and continue the session without research-backed data, noting the limitation to the user.

- **FR-013**: Research findings SHALL inform the brainstorming Q&A. When presenting alternatives, the agent SHALL include research-backed pros/cons.
  - **Error**: If no alternatives were found by research, the agent SHALL present user-provided alternatives and note "No research data available for comparison."

- **FR-014**: The Brainstorming Agent SHALL produce a brief with the same enriched format as the Ideation Agent (FR-010).
  - **Error**: If no research was performed during the session, the Research Findings section SHALL state "No research performed" and the Risk Assessment section SHALL state "Not assessed."

---

## 5. User Stories

### US-01 -- Research-Backed Ideation (Priority: P1) MVP

**As a** Human Developer, **I want** the Ideation Agent to research my idea before asking questions, **so that** the resulting brief is grounded in real-world context.

**Why P1**: Research-backed briefs produce better specs. Without research, briefs are based solely on the user's knowledge.

**Independent Test**: Describe an idea to the Ideation Agent. Verify: the resulting brief contains a "Research Findings" section with at least 2 cited sources.

**Acceptance Scenarios**:
1. **Given** a user describes "build a CLI tool for database migrations," **When** the Ideation Agent processes the idea, **Then** it dispatches the Research Skill, and the brief includes competitive analysis of existing migration tools.
2. **Given** the Research Skill finds 3 competing tools, **When** the brief is written, **Then** each competitor has a source URL.

---

### US-02 -- Technology Evaluation in Brainstorming (Priority: P2)

**As a** Human Developer, **I want** the Brainstorming Agent to research technologies before recommending them, **so that** recommendations are based on current data.

**Why P2**: Technology recommendations without research may reference outdated or abandoned packages.

**Independent Test**: Ask the Brainstorming Agent to compare two frameworks. Verify: comparison includes version numbers, maintenance status, and source URLs.

**Acceptance Scenarios**:
1. **Given** a brainstorming session exploring "React vs Svelte for the UI," **When** the agent researches both, **Then** findings include latest versions, GitHub stars, and last release dates with source URLs.

---

### Edge Cases

- What happens when web research is unavailable (network issues)? The skill writes "Web research unavailable" and proceeds with codebase-only findings.
- What happens when the codebase is empty (new project)? Codebase scope returns "No existing code found" and the skill continues with other scopes.
- What happens when the research topic is too broad? The skill focuses on the specific questions provided rather than trying to cover the entire topic.

---

## 6. User Flows

### 6.1 Ideation with Research

1. User describes idea to Ideation Agent.
2. Ideation Agent dispatches Research Skill with scope [web, codebase].
3. Research Skill searches web for competitive landscape.
4. Research Skill searches codebase for relevant existing code.
5. Research Skill writes findings to temporary file.
6. Ideation Agent reads research findings.
7. Ideation Agent asks clarifying questions (informed by research).
8. User answers questions.
9. Ideation Agent writes brief with Research Findings section.
10. Brief includes cited sources.

### 6.2 Brainstorming with Technology Research

1. Brainstorming session reaches a technology decision point.
2. Agent dispatches Research Skill with scope [web, packages].
3. Research Skill evaluates packages on registries.
4. Research Skill writes comparison to file.
5. Agent presents research-backed alternatives to user.
6. User makes informed decision.
7. Decision is recorded in brief with research evidence.

---

## 7. Data Model

### 7.1 Research Request

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| topic | string | yes | 1-200 characters |
| scope | array of string | yes | Each element one of: "web", "codebase", "packages". At least 1 element. |
| questions | array of string | yes | 1-10 items, each 1-500 characters |
| output_file | string | yes | Valid file path |

### 7.2 Research Output Sections

| Section | Content | Source Scope |
|---------|---------|-------------|
| Questions & Answers | Direct answers to each question with sources | All scopes |
| Competitive/Analogous Solutions | Table of alternatives | web |
| Technology Evaluation | Table of tech with versions and status | web, packages |
| Codebase Context | Relevant existing code findings | codebase |
| Risks & Concerns | Identified risks from research | All scopes |

---

## 8. API / Interface Design

### 8.1 Research Skill Invocation

**Dispatch**: Via `runSubagent` from any coordinator agent.

**Prompt template**:
```
Research the following topic and write findings to {output_file}.

Topic: {topic}
Scope: {scope}
Questions:
1. {question_1}
2. {question_2}
...
```

**Output**: Structured markdown file at the specified path.

---

## 9. Architecture

### 9.1 Directory Structure

```
.github/
  skills/
    research/SKILL.md           # NEW: shared research skill
  agents/
    ideation.agent.md           # MODIFIED: dispatches research skill
    brainstorming.agent.md      # MODIFIED: dispatches research skill
```

### 9.2 Key Design Decisions

**Decision 1: Shared skill, not per-agent research**
- **Rationale**: Both Ideation and Brainstorming need the same research capabilities. A shared skill avoids duplication.
- **Alternatives**: Each agent has its own research logic, new Research Agent.
- **Consequences**: One skill to maintain. Any agent can use it.

**Decision 2: File-based output (not inline)**
- **Rationale**: Research findings may be large. Writing to a file lets the invoking agent read selectively and avoids prompt bloat.
- **Alternatives**: Return findings inline in the subagent response.
- **Consequences**: Requires file I/O, but keeps context windows clean.

**Decision 3: Enriched brief format**
- **Rationale**: The Spec Architect V2 benefits from research-backed briefs. Adding Research Findings, Risk Assessment, and Technical Feasibility sections directly improves spec quality.
- **Alternatives**: Keep current brief format.
- **Consequences**: Slightly longer briefs, but significantly richer input for specification.

---

## 10. Non-Functional Requirements

### 10.1 Performance
- Research Skill SHALL complete within 5 minutes per invocation.
- Individual web fetches SHALL timeout after 30 seconds.

### 10.2 Security
- The Research Skill SHALL NOT execute code found on the web.
- The Research Skill SHALL NOT store credentials or API keys.
- Web fetch results SHALL be treated as untrusted input.

---

## 11. Test Requirements

### 11.1 Test Approach

Testing focuses on BDD acceptance tests verifying the Research Skill's output format and the enriched brief output from both agents. Unit-level testing is not applicable since the skill is a prompt-driven markdown file with no executable code. Validation SHALL be performed by inspecting output files for structural compliance and source citation presence.

### 11.2 BDD / Acceptance Tests

```gherkin
Feature: Research Skill

  Scenario: Web research for competitive analysis
    Given a research request with topic "database migration CLI" and scope ["web"]
    When the Research Skill executes
    Then the output file contains a Competitive/Analogous Solutions table
    And each entry has a source URL

  Scenario: Codebase research
    Given a research request with scope ["codebase"]
    And the workspace contains relevant code
    When the Research Skill executes
    Then the output file contains a Codebase Context section

  Scenario: Web fetch timeout
    Given a web source is unavailable
    When the Research Skill attempts to fetch it
    Then it skips the source with a note "unavailable"
    And continues with remaining sources

Feature: Ideation with Research

  Scenario: Brief includes research findings
    Given the user describes an idea
    When the Ideation Agent completes the brief
    Then the brief contains a Research Findings section
    And at least 2 sources are cited

Feature: Brainstorming with Research

  Scenario: Technology comparison backed by research
    Given a brainstorming session reaches a technology decision
    When the agent dispatches the Research Skill
    Then alternatives are presented with version numbers and status
    And sources are cited
```

---

## 12. Constraints & Assumptions

### Constraints
- Web research depends on `fetch_webpage` tool availability.
- Package registry lookups depend on public registry APIs being accessible.
- The Research Skill cannot access private/authenticated APIs.

### Assumptions
1. `fetch_webpage` works reliably for major documentation sites, GitHub, npm, and PyPI.
2. The Ideation and Brainstorming agents are refactored to use the coordinator + skills pattern (or at minimum, can dispatch subagents).
3. Research output files are temporary; they are consumed by the invoking agent and do not need to be committed.

---

## 13. Out of Scope

- **Research caching**: Results are not cached across invocations. Each research request starts fresh.
- **Authenticated API access**: No support for APIs requiring credentials.
- **Research for other agents**: While the skill is technically usable by any agent, only Ideation and Brainstorming are specified here.
- **Pipeline analytics**: Tracking research quality metrics is future work.

---

## 14. Open Questions

None remaining.

---

## 15. Glossary

- **Research Skill**: A reusable skill that performs structured research across web, codebase, and package registries.
- **Enriched brief**: A brief output format that includes Research Findings, Risk Assessment, and Technical Feasibility sections beyond the standard brief format.

---

## 16. Traceability Matrix

| FR ID | Requirement Summary | User Story | Acceptance Scenario | Test Type | Test Section Ref |
|-------|-------------------|------------|--------------------|-----------|----|
| FR-001 | Research Skill at .github/skills/research/ | US-01, US-02 | All | BDD | 11.2 |
| FR-003 | Web scope research | US-01 | Scenario 1 | BDD | 11.2 |
| FR-004 | Codebase scope research | US-01 | Scenario 2 | BDD | 11.2 |
| FR-005 | Packages scope research | US-02 | Scenario 1 | BDD | 11.2 |
| FR-006 | Structured output format | US-01, US-02 | All | BDD | 11.2 |
| FR-002 | Research request parameters | US-01, US-02 | All | BDD | 11.2 |
| FR-007 | 5-minute completion timeout | US-01, US-02 | Timeout Scenario | BDD | 11.2 |
| FR-008 | Ideation dispatches Research Skill | US-01 | Scenario 1, 2 | BDD | 11.2 |
| FR-009 | Brief includes Research Findings | US-01 | Brief Scenario | BDD | 11.2 |
| FR-010 | Enriched brief format sections | US-01 | Brief Scenario | BDD | 11.2 |
| FR-011 | Source citations in brief | US-01 | Scenario 2 | BDD | 11.2 |
| FR-012 | Brainstorming dispatches Research Skill | US-02 | Scenario 1 | BDD | 11.2 |
| FR-013 | Research-backed pros/cons | US-02 | Scenario 1 | BDD | 11.2 |
| FR-014 | Brainstorming enriched brief format | US-02 | Brief Scenario | BDD | 11.2 |

---

## 17. Technical References

- Write the Docs - Research in Documentation, https://www.writethedocs.org/guide/, consulted 2026-04-05

---

## 18. Version History

| Version | Date | Author | Summary of Changes |
|---------|------|--------|--------------------|
| 1.0 | 2026-04-05 | Spec Architect | Initial specification |
| 1.1 | 2026-04-05 | Spec Architect | Validation pass: added error behaviors to FR-008/009/011/012/013/014, completed traceability matrix, added Section 11.1, fixed ambiguous word, created companion artifacts |
