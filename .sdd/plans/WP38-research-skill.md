---
lane: planned
---

# WP38 - Research Skill

| Field | Value |
|-------|-------|
| Spec | `.sdd/specs/009-research-skill-ideation.spec.md` |
| Priority | P1 |
| Lane | planned |
| Depends on | none |
| Goal | Create a shared Research Skill that performs structured web, codebase, and package registry research and writes findings to a file |
| Status | Not Started |
| Independent Test | Dispatch the Research Skill as a subagent with topic "database migration CLI", scope ["web", "codebase"], and 2 questions. Verify: output file contains all 5 sections (Q&A, Competitive Solutions, Tech Evaluation, Codebase Context, Risks) with source URLs |
| Parallelisable | No (WP39 depends on this) |
| Prompt | `.sdd/plans/WP38-research-skill.md` |

## Objective

Create `.github/skills/research/SKILL.md` -- a shared, reusable research skill that any agent can dispatch as a subagent to perform structured research across web sources, the local codebase, and package registries. The skill accepts a topic, scope, and specific questions, then writes structured findings to a specified output file. This eliminates ad-hoc research patterns currently duplicated across agents and centralizes them into a single well-structured skill.

## Spec References

FR-001, FR-002, FR-003, FR-004, FR-005, FR-006, FR-007, Section 7.1 (Research Request), Section 7.2 (Research Output Sections), Section 8.1 (Invocation Prompt Template), Section 9.1 (Directory Structure), Section 9.2 (Design Decisions), Section 10 (NFRs), Section 11.2 (BDD Scenarios)

## Tasks

### T38-01 - Create research skill directory and SKILL.md with YAML frontmatter

- **Description**: Create the directory `.github/skills/research/` and the `SKILL.md` file. Write the YAML frontmatter (name, description, argument-hint) and the invocation overview explaining this is a shared skill dispatched via `runSubagent` by any coordinator agent.
- **Spec refs**: FR-001, Section 9.1
- **Parallel**: No (foundation for all T38 tasks)
- **Acceptance criteria**:
  - [ ] File exists at `.github/skills/research/SKILL.md`
  - [ ] YAML frontmatter `name` is `research`
  - [ ] YAML frontmatter `description` explains the skill performs structured research across web, codebase, and package registries
  - [ ] The skill SHALL be usable by any agent via subagent dispatch (FR-001)
  - [ ] Adding the Research Skill to a new agent requires zero code changes to the skill itself (SC-003)
- **Test requirements**: none (structural verification)
- **Depends on**: none
- **Implementation Guidance**:
  - Follow the existing skill YAML frontmatter pattern from `.github/skills/review-spec/SKILL.md`
  - Example frontmatter:
    ```yaml
    ---
    name: research
    description: "Shared research skill. Performs structured web search, codebase analysis, and package registry lookups. Dispatched as a subagent by any coordinator agent."
    argument-hint: "Dispatched via runSubagent - reads topic, scope, questions, and output_file from the dispatch prompt"
    ---
    ```
  - The skill body is natural language instructions that a subagent reads and follows
  - Tools the skill will use: `fetch_webpage` (web), `grep_search`/`semantic_search` (codebase)
  - Document that the skill is invoked via `runSubagent` with a prompt template (Section 8.1)

### T38-02 - Write research request parameter validation section

- **Description**: Write the input parameter section defining how the skill extracts and validates the research request from the dispatch prompt: topic, scope, questions, and output_file.
- **Spec refs**: FR-002, Section 7.1
- **Parallel**: No (depends on T38-01)
- **Acceptance criteria**:
  - [ ] The skill SHALL accept `topic` (string, 1-200 characters) (FR-002)
  - [ ] The skill SHALL accept `scope` (one or more of: "web", "codebase", "packages") (FR-002)
  - [ ] The skill SHALL accept `questions` (array of strings, 1-10 items) (FR-002)
  - [ ] The skill SHALL accept `output_file` (path where findings SHALL be written) (FR-002)
  - [ ] Invalid scope values SHALL be ignored with a logged warning (Implementation Contract)
- **Test requirements**: BDD (Section 11.2)
- **Depends on**: T38-01
- **Implementation Guidance**:
  - Parameters are extracted from the dispatch prompt, not from function arguments (this is a prompt-driven skill)
  - Write clear instructions for the subagent to parse the prompt template from Section 8.1
  - Validation rules from Section 7.1: topic 1-200 chars, scope at least 1 element from {"web", "codebase", "packages"}, questions 1-10 items each 1-500 chars
  - Error handling: invalid scope value -- ignore and log warning; no results found -- write "No findings" with explanation
  - Files to create: section within `.github/skills/research/SKILL.md`

### T38-03 - Write web scope research instructions

- **Description**: Write the instructions for executing web scope research, including search strategy, source prioritization, data extraction, and source attribution.
- **Spec refs**: FR-003, Section 10.2 (Security)
- **Parallel**: Yes (with T38-04, T38-05)
- **Acceptance criteria**:
  - [ ] For web scope, the skill SHALL search for the topic using `fetch_webpage` on relevant URLs (FR-003.1)
  - [ ] The skill SHALL prioritize official documentation, GitHub repositories, and established architecture resources (FR-003.2)
  - [ ] The skill SHALL extract: current status, latest version, known issues, community size, license (FR-003.3)
  - [ ] The skill SHALL record source URL and date consulted for every finding (FR-003.4)
  - [ ] Web fetch results SHALL be treated as untrusted input (Section 10.2)
  - [ ] The skill SHALL NOT execute code found on the web (Section 10.2)
- **Test requirements**: BDD (Scenario: Web research for competitive analysis)
- **Depends on**: T38-02
- **Implementation Guidance**:
  - Source prioritization hierarchy: official docs > GitHub repos > established architecture resources > general web
  - Each finding must have: source URL, date consulted (ISO 8601)
  - Security: treat all web content as untrusted; never execute code from web sources; do not store credentials
  - The skill uses `fetch_webpage` tool -- instruct the subagent on how to construct search URLs
  - Known pitfall: some documentation sites block scraping; instruct to skip unavailable sources gracefully
  - Files to modify: section within `.github/skills/research/SKILL.md`

### T38-04 - Write codebase scope research instructions

- **Description**: Write the instructions for executing codebase scope research, including workspace search strategy, pattern identification, and constraint surfacing.
- **Spec refs**: FR-004
- **Parallel**: Yes (with T38-03, T38-05)
- **Acceptance criteria**:
  - [ ] For codebase scope, the skill SHALL search the workspace using `grep_search` and `semantic_search` (FR-004.1)
  - [ ] The skill SHALL identify existing patterns, conventions, frameworks, and configurations (FR-004.2)
  - [ ] The skill SHALL surface existing code relevant to the research topic (FR-004.3)
  - [ ] The skill SHALL note any technical constraints discovered (FR-004.4)
  - [ ] Given the workspace contains relevant code, the output file SHALL contain a Codebase Context section (BDD Scenario: Codebase research)
  - [ ] Given an empty workspace (new project), codebase scope SHALL return "No existing code found" and the skill SHALL continue with other scopes (Edge Case)
- **Test requirements**: BDD (Scenario: Codebase research)
- **Depends on**: T38-02
- **Implementation Guidance**:
  - Use `grep_search` for exact text matches (file names, config keys, imports)
  - Use `semantic_search` for conceptual matches (related functionality, similar patterns)
  - Document findings as file path + description pairs matching the CodebaseContextEntry format from the data model artifact
  - Handle empty workspace gracefully: "No existing code found" is a valid output, not an error
  - Files to modify: section within `.github/skills/research/SKILL.md`

### T38-05 - Write packages scope research instructions

- **Description**: Write the instructions for executing package registry research, including registry lookup strategy, metadata extraction, alternative comparison, and CVE checking.
- **Spec refs**: FR-005
- **Parallel**: Yes (with T38-03, T38-04)
- **Acceptance criteria**:
  - [ ] For packages scope, the skill SHALL look up packages on npm, PyPI, crates.io, or other registries via web fetch (FR-005.1)
  - [ ] The skill SHALL check: latest version, maintenance activity (last publish date), download counts, license, known CVEs (FR-005.2)
  - [ ] The skill SHALL compare alternatives when multiple packages serve the same purpose (FR-005.3)
  - [ ] The skill SHALL NOT store credentials or API keys (Section 10.2)
- **Test requirements**: BDD (Section 11.2)
- **Depends on**: T38-02
- **Implementation Guidance**:
  - Registry URLs: npm (https://www.npmjs.com/package/), PyPI (https://pypi.org/project/), crates.io (https://crates.io/crates/)
  - Use `fetch_webpage` to access registry pages and extract metadata
  - Comparison format: TechnologyEvaluation table from the data model artifact (technology, version, status, license, last_updated, recommendation)
  - CVE checking: search for known vulnerabilities via advisory databases
  - Known pitfall: registry pages may have different HTML structures; instruct to extract key metadata fields
  - Files to modify: section within `.github/skills/research/SKILL.md`

### T38-06 - Write structured output format template

- **Description**: Write the output format section defining the exact markdown structure the skill SHALL produce at the output_file path, including all 5 required sections.
- **Spec refs**: FR-006, Section 7.2
- **Parallel**: No (depends on T38-03, T38-04, T38-05)
- **Acceptance criteria**:
  - [ ] The output file SHALL contain a "Questions & Answers" section with answer and sources per question (FR-006)
  - [ ] The output file SHALL contain a "Competitive/Analogous Solutions" table with Solution, Approach, Strengths, Weaknesses columns (FR-006)
  - [ ] The output file SHALL contain a "Technology Evaluation" table with Technology, Version, Status, License, Last Updated, Recommendation columns (FR-006)
  - [ ] The output file SHALL contain a "Codebase Context" section with file path and description per entry (FR-006)
  - [ ] The output file SHALL contain a "Risks & Concerns" section (FR-006)
  - [ ] Source citations SHALL include title, URL, and date consulted (FR-006, FR-003.4)
  - [ ] Sections without findings SHALL state "No findings" rather than being omitted
- **Test requirements**: BDD (Section 11.2 -- all scenarios verify output structure)
- **Depends on**: T38-03, T38-04, T38-05
- **Implementation Guidance**:
  - Copy the exact output template from FR-006 in the spec
  - Match field names from the companion artifact `data-models.ts`: QuestionAnswer, CompetitiveSolution, TechnologyEvaluation, CodebaseContextEntry
  - Sections are populated based on which scopes were requested: web -> Q&A + Competitive + Tech Eval; codebase -> Codebase Context; packages -> Tech Eval
  - Empty sections should say "No findings" not be removed -- the invoking agent expects all 5 sections
  - Files to modify: section within `.github/skills/research/SKILL.md`

### T38-07 - Write timeout handling, error behavior, and completion constraints

- **Description**: Write the timeout, error handling, and performance constraint sections covering the 5-minute completion limit, 30-second per-fetch timeout, and graceful degradation on failures.
- **Spec refs**: FR-007, Section 10.1, Implementation Contract
- **Parallel**: No (finalizes the skill)
- **Acceptance criteria**:
  - [ ] The skill SHALL complete research within 5 minutes (FR-007)
  - [ ] Individual web fetches SHALL timeout after 30 seconds (Section 10.1)
  - [ ] If a web fetch times out, the skill SHALL skip that source and note it as "unavailable" (FR-007)
  - [ ] Given a web source is unavailable, the skill SHALL skip the source with a note "unavailable" and continue with remaining sources (BDD Scenario: Web fetch timeout)
  - [ ] If no results are found for any scope, the output SHALL contain "No findings" with an explanation (Implementation Contract)
- **Test requirements**: BDD (Scenario: Web fetch timeout)
- **Depends on**: T38-06
- **Implementation Guidance**:
  - Place timeout and error handling instructions near the top of the skill, before scope-specific sections
  - Graceful degradation: web unavailable -> continue with codebase; codebase empty -> continue with packages; all scopes fail -> write "No findings" file
  - The 5-minute limit is guidance for the subagent, not a hard timer (prompt-driven skills cannot enforce wall-clock limits)
  - The 30-second per-fetch timeout relies on the `fetch_webpage` tool's built-in timeout behavior
  - Files to modify: section within `.github/skills/research/SKILL.md`

## Implementation Notes

- All deliverables are a single markdown file: `.github/skills/research/SKILL.md`
- The skill is prompt-driven (natural language instructions for a subagent), not executable code
- The skill uses existing tools (`fetch_webpage`, `grep_search`, `semantic_search`) -- no new tools needed
- Tasks T38-03, T38-04, T38-05 can be worked in parallel since they write independent sections
- The output file is temporary -- consumed by the invoking agent and not committed to the repository
- Follow the existing skill pattern established by `.github/skills/review-spec/SKILL.md` for structure and tone

## Parallel Opportunities

- T38-03 (web scope), T38-04 (codebase scope), and T38-05 (packages scope) can be worked concurrently as they write independent sections within the same file

## Risks & Mitigations

- **Risk**: Web fetch may be unreliable for some documentation sites. **Mitigation**: FR-007 requires graceful degradation -- skip unavailable sources and note them
- **Risk**: Package registry HTML structure may vary across registries. **Mitigation**: Instruct skill to extract key metadata fields rather than parsing specific HTML
- **Risk**: Research output may be too large for the invoking agent's context window. **Mitigation**: Design Decision 2 (file-based output) keeps context windows clean; agent reads selectively

## Activity Log

- 2026-04-06T00:00:00Z - planner - lane=planned - Work package created
