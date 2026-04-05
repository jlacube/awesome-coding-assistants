# Docs Agent - Skill-Based Documentation Generation -- Specification

> **Source brief**: `.sdd/ideas/002-sdd-pipeline-v2-universal-skill-architecture.md`
> **Feature branch**: `007-docs-agent`
> **Status**: Validated
> **Version**: 1.0

---

## 1. Overview

Introduce a new Docs Agent to the SDD pipeline: a skill-based coordinator that dispatches 6 sequential skills (architecture docs, API reference, user guide, developer guide, changelog, inline code docs) to produce and maintain living documentation in `.sdd/docs/`. The Docs Agent runs after every work package is reviewed and approved (lane = done). This separates documentation from the Coder's responsibilities, ensuring doc quality gets dedicated attention rather than being a secondary concern during implementation.

---

## 2. Goals & Success Criteria

- **SC-001**: Documentation is produced after every approved WP, keeping docs in sync with implementation. Verified by: `.sdd/docs/` is updated after every WP whose lane transitions to `done`.
- **SC-002**: API reference documentation is generated from contract files, ensuring accuracy. Verified by: API docs match contract schemas field-by-field.
- **SC-003**: Each doc skill runs in a fresh context window for focused generation. Verified by: each subagent loads only its SKILL.md plus the relevant source material.
- **SC-004**: Adding a new documentation dimension requires creating one skill file. Verified by: no coordinator edit needed.

---

## 3. Users & Roles

- **Human Developer (primary reader)**: Reads generated documentation for understanding, onboarding, and reference.
- **Review Coordinator (invoker)**: Triggers the Docs Agent after approving a WP (setting lane = done).
- **Orchestrator Agent (invoker)**: Triggers the Docs Agent as part of the pipeline after review approval.
- **Coder Agent (no longer responsible)**: The Coder no longer maintains `.sdd/docs/`. That responsibility transfers to the Docs Agent.

---

## 4. Functional Requirements

### 4.1 Docs Agent Coordinator

#### 4.1.1 Trigger and Context

- **FR-001**: The Docs Agent SHALL be triggered after a WP is reviewed and approved (lane = done). The trigger SHALL include:
  1. The approved WP file path
  2. The spec file path
  3. The contract files directory for the WP
  4. The implementation source files modified by the WP
  - Error: If no WP path is provided, the coordinator SHALL halt with a descriptive error message.

- **FR-002**: The coordinator SHALL read:
  1. The approved WP file and its task list
  2. The spec referenced by the WP
  3. Contract files in `.sdd/plans/contracts/<WP-slug>/`
  4. Implementation source files (from git diff of the WP's commits)
  5. Existing documentation in `.sdd/docs/` (to update, not recreate)
  - Error: If a referenced file does not exist, the coordinator SHALL log a warning and proceed with available files. If the WP file itself is missing, the coordinator SHALL halt.

#### 4.1.2 Dynamic Skill Discovery

- **FR-003**: The coordinator SHALL discover doc skills by scanning `.github/skills/doc-*/SKILL.md`.
  - Error: If zero skills found, halt and report.

- **FR-004**: The coordinator SHALL dispatch skills in canonical order:
  1. `doc-architecture` (architecture overview, component diagrams, design decisions)
  2. `doc-api-reference` (API endpoint documentation from contracts)
  3. `doc-user-guide` (end-user documentation for features)
  4. `doc-developer-guide` (setup, conventions, contributing)
  5. `doc-changelog` (changelog entry for the WP)
  6. `doc-inline-code` (code comments and docstrings in source files)

#### 4.1.3 Skill Dispatch

- **FR-005**: Each skill SHALL be dispatched as a subagent with:
  1. The skill file path
  2. The WP file path and task list
  3. The spec path and contract files
  4. Implementation source files
  5. Existing docs directory for incremental updates
  6. Active doc-domain patterns

- **FR-006**: Skills SHALL execute sequentially, each reading existing docs before writing updates. Skills update existing doc files incrementally -- they do NOT recreate docs from scratch each time.

- **FR-007**: If a skill fails, the coordinator SHALL log the failure and continue to the next skill. Doc generation is best-effort; a failed changelog does not prevent API docs.

#### 4.1.4 Patterns Consumption

- **FR-008**: The coordinator SHALL read `.sdd/reviews/doc-patterns.md` (if exists) before dispatching skills.

#### 4.1.5 Commit Policy

- **FR-009**: After all skills complete, the coordinator SHALL commit documentation changes:
  ```
  git add .sdd/docs/ <modified source files for inline docs>
  git commit -m "docs(docs): update documentation for WP<NN>"
  ```
  - Error: If no documentation files were modified (all skills produced no output), the coordinator SHALL skip the commit and log that no updates were needed. If the git commit fails, the coordinator SHALL report the error to the invoker.

#### Implementation Contract -- Docs Agent Coordinator

**Inputs**: Approved WP path, spec path, contracts directory, implementation files.
**Outputs**: Updated `.sdd/docs/` files, updated source file docstrings.
**Error behaviors**: Zero skills - halt. Skill failure - log and continue. No WP provided - halt.

---

### 4.2 Documentation Skills

> All skill FRs (FR-010 through FR-020) inherit the error behavior of FR-007: if a skill fails, the coordinator logs the failure and continues to the next skill.

#### 4.2.1 Architecture Docs Skill (doc-architecture)

- **FR-010**: SHALL produce/update `.sdd/docs/architecture.md` with:
  1. System overview (from spec Section 9.1)
  2. Component diagram (Mermaid or prose)
  3. Technology stack summary (from spec Section 9.2)
  4. Key design decisions (from spec Section 9.4)
  5. Directory structure (from actual codebase, not spec)
  6. Data flow descriptions

- **FR-011**: On incremental updates, the skill SHALL update affected sections without overwriting unrelated sections.

#### 4.2.2 API Reference Skill (doc-api-reference)

- **FR-012**: SHALL produce/update `.sdd/docs/api-reference.md` with:
  1. One section per API endpoint
  2. Method, path, description
  3. Request parameters and body (from contracts)
  4. Response schema (from contracts)
  5. Error codes and meanings (from error catalog contract)
  6. Authentication requirements
  7. Example request/response (generated from schemas)

- **FR-013**: API docs SHALL be generated from contract files (`api-contracts.<ext>`, `error-catalog.<ext>`), NOT from prose interpretation. If contracts exist, they are the source of truth.
  - Error: If no contract files exist for the WP, the skill SHALL skip API doc generation and log that no contracts were found.

#### 4.2.3 User Guide Skill (doc-user-guide)

- **FR-014**: SHALL produce/update `.sdd/docs/user-guide.md` with:
  1. Feature descriptions (from user stories)
  2. Step-by-step usage instructions
  3. Configuration options (from config schema contract)
  4. Common workflows
  5. Troubleshooting for expected error scenarios

#### 4.2.4 Developer Guide Skill (doc-developer-guide)

- **FR-015**: SHALL produce/update `.sdd/docs/developer-guide.md` with:
  1. Development environment setup
  2. Project structure overview
  3. Coding conventions in use
  4. Testing approach and commands
  5. How to add new features (following the project's patterns)

#### 4.2.5 Changelog Skill (doc-changelog)

- **FR-016**: SHALL append to `.sdd/docs/CHANGELOG.md`:
  1. WP identifier and title
  2. Date
  3. List of changes (from WP task list and descriptions)
  4. Breaking changes (if any)
  5. Dependencies added/changed

- **FR-017**: Changelog entries SHALL be prepended (newest first), not appended.

#### 4.2.6 Inline Code Docs Skill (doc-inline-code)

- **FR-018**: SHALL add/update docstrings and comments in implementation source files:
  1. Module-level docstrings describing purpose
  2. Function/method docstrings with parameter descriptions, return types, raises/throws
  3. Complex logic comments explaining "why", not "what"
  4. Type annotations (if missing and language supports them)

- **FR-019**: The skill SHALL NOT modify implementation logic; only add documentary content.

- **FR-020**: The skill SHALL follow the project's existing docstring convention (if one exists). If none exists, use the language's standard (Python: Google style, TypeScript: JSDoc, Go: godoc, Rust: rustdoc).

---

## 5. User Stories

### US-01 -- Automated Docs After WP Approval (Priority: P1) MVP

**As a** Human Developer, **I want** documentation automatically updated after every approved WP, **so that** docs stay current without manual effort.

**Why P1**: Stale docs are a top developer complaint. Automation eliminates the problem.

**Independent Test**: Approve WP03. Verify: `.sdd/docs/` files are updated to reflect WP03's changes.

**Acceptance Scenarios**:
1. **Given** WP03 is approved (lane = done), **When** the Docs Agent runs, **Then** architecture.md, api-reference.md, changelog, and developer-guide.md are updated.
2. **Given** WP03 adds 2 new API endpoints, **When** doc-api-reference runs, **Then** 2 new endpoint sections appear in api-reference.md with request/response schemas from contracts.

---

### US-02 -- Contract-Based API Docs (Priority: P1) MVP

**As a** Human Developer, **I want** API reference docs generated from contract files, **so that** docs exactly match the implementation contracts.

**Why P1**: API docs derived from contracts are always accurate.

**Independent Test**: Compare api-reference.md field names against contract files. Verify: zero mismatches.

**Acceptance Scenarios**:
1. **Given** `api-contracts.ts` defines POST /users with `CreateUserInput { email, name, role }`, **When** doc-api-reference runs, **Then** api-reference.md shows POST /users with fields email, name, role.

---

### Edge Cases

- What happens when this is the first WP (no existing docs)? Skills create the doc files from scratch.
- What happens when a WP changes no API endpoints? doc-api-reference has no updates and exits cleanly.
- What happens when source files have existing docstrings? doc-inline-code updates them if they are stale, leaves them if accurate.

---

## 6. User Flows

### 6.1 Post-Approval Documentation Flow

1. Review Coordinator approves WP (sets lane = done).
2. Orchestrator triggers Docs Agent with WP context.
3. Coordinator reads WP, spec, contracts, implementation files, existing docs.
4. Coordinator reads doc-patterns.md.
5. Coordinator discovers doc skills.
6. Dispatch doc-architecture: update architecture overview.
7. Dispatch doc-api-reference: update API docs from contracts.
8. Dispatch doc-user-guide: update user-facing docs.
9. Dispatch doc-developer-guide: update dev setup and conventions.
10. Dispatch doc-changelog: prepend changelog entry.
11. Dispatch doc-inline-code: update source file docstrings.
12. Coordinator commits all doc changes.

---

## 7. Data Model

### 7.1 Documentation Files

| File | Skill | Content |
|------|-------|---------|
| `.sdd/docs/architecture.md` | doc-architecture | System design, components, decisions |
| `.sdd/docs/api-reference.md` | doc-api-reference | Endpoint docs from contracts |
| `.sdd/docs/user-guide.md` | doc-user-guide | Feature usage instructions |
| `.sdd/docs/developer-guide.md` | doc-developer-guide | Dev setup, conventions |
| `.sdd/docs/CHANGELOG.md` | doc-changelog | Version history entries |
| Source files (*.ts, *.py, *.go, *.rs) | doc-inline-code | Docstrings and comments |

---

## 8. API / Interface Design

### 8.1 Coordinator Invocation

**Invocation**: Triggered by Orchestrator or Review Coordinator after WP approval.

**Prompt**:
```
WP<NN> has been approved.
WP file: <wp_path>
Spec: <spec_path>
Contracts: <contracts_dir>
Implementation files: <file_list>
Update documentation in .sdd/docs/.
```

---

## 9. Architecture

### 9.1 Directory Structure

```
.github/
  agents/
    docs-agent.agent.md              # NEW: documentation coordinator
  skills/
    doc-architecture/SKILL.md        # NEW
    doc-api-reference/SKILL.md       # NEW
    doc-user-guide/SKILL.md          # NEW
    doc-developer-guide/SKILL.md     # NEW
    doc-changelog/SKILL.md           # NEW
    doc-inline-code/SKILL.md         # NEW

.sdd/
  docs/
    architecture.md
    api-reference.md
    user-guide.md
    developer-guide.md
    CHANGELOG.md
```

### 9.2 Key Design Decisions

**Decision 1: Dedicated Docs Agent (not part of Coder)**
- **Rationale**: User chose to separate concerns. Coder focuses on code; Docs Agent focuses on documentation quality.
- **Alternatives**: Coder updates docs, auto-generate only, Reviewer triggers updates.
- **Consequences**: Documentation gets dedicated attention. Coder is simpler.

**Decision 2: Run after every WP approval (not batch)**
- **Rationale**: User chose incremental updates. Docs stay current WP-by-WP rather than getting a bulk update at the end.
- **Alternatives**: Once all WPs done, on-demand only.
- **Consequences**: More frequent doc updates. Each update is smaller and more focused.

---

## 10. Non-Functional Requirements

- **NFR-001**: Full doc generation (6 skills) SHALL complete within 20 minutes per WP.
- **NFR-002**: Skill failure SHALL NOT block other skills (best-effort).

---

## 11. Test Requirements

### 11.2 BDD / Acceptance Tests

```gherkin
Feature: Docs Agent - Post-Approval Documentation

  Scenario: Generate docs after WP approval
    Given WP03 is approved with 2 new API endpoints
    When the Docs Agent runs
    Then architecture.md is updated
    And api-reference.md has 2 new endpoint sections
    And CHANGELOG.md has a new entry for WP03
    And developer-guide.md is updated

  Scenario: API docs match contracts
    Given api-contracts.ts defines POST /users with CreateUserInput
    When doc-api-reference runs
    Then api-reference.md shows POST /users with CreateUserInput fields

  Scenario: Skill failure does not block others
    Given doc-changelog encounters an error
    When the coordinator detects the failure
    Then it logs the error
    And continues to doc-inline-code
```

---

## 12. Constraints & Assumptions

### Constraints
- Must operate within VS Code Copilot Chat agent framework.
- Skills execute sequentially.

### Assumptions
1. Coder V2 (Spec 004) is implemented and no longer updates `.sdd/docs/`.
2. Contract files exist in `.sdd/plans/contracts/`.
3. The Review Coordinator or Orchestrator triggers the Docs Agent after WP approval.

---

## 13. Out of Scope

- **Code implementation**: Docs Agent produces documentation, not code (except docstrings).
- **External publishing**: Docs are markdown files in `.sdd/docs/`; publishing to a website is separate.
- **Real-time doc previews**: No live preview server.

---

## 14. Open Questions

None remaining.

---

## 15. Glossary

- **Living documentation**: Docs that are updated incrementally as the codebase evolves, never becoming stale.
- **Contract-based docs**: Documentation generated from language-specific contract files rather than prose interpretation.

---

## 16. Traceability Matrix

| FR ID | Requirement Summary | User Story | Acceptance Scenario | Test Type | Test Section Ref |
|-------|-------------------|------------|--------------------|-----------|----|
| FR-001 | Trigger after WP approval | US-01 | Scenario 1 | BDD | 11.2 |
| FR-002 | Read WP, spec, contracts, source, existing docs | US-01 | Scenario 1 | BDD | 11.2 |
| FR-003 | Dynamic skill discovery | US-01 | Scenario 1 | BDD | 11.2 |
| FR-004 | Dispatch skills in canonical order | US-01 | Scenario 1 | BDD | 11.2 |
| FR-005 | Skill dispatch parameters | US-01 | Scenario 1 | BDD | 11.2 |
| FR-006 | Sequential skill dispatch | US-01 | Scenario 1 | BDD | 11.2 |
| FR-007 | Skill failure tolerance | US-01 | BDD Scenario 3 | BDD | 11.2 |
| FR-008 | Read doc-patterns before dispatch | US-01 | Scenario 1 | BDD | 11.2 |
| FR-009 | Commit after all skills complete | US-01 | Scenario 1 | BDD | 11.2 |
| FR-010 | Architecture docs generation | US-01 | Scenario 1 | BDD | 11.2 |
| FR-011 | Incremental section updates | US-01 | Scenario 1 | BDD | 11.2 |
| FR-012 | API reference from contracts | US-02 | Scenario 1, BDD Scenario 2 | BDD | 11.2 |
| FR-013 | Contract-based API docs | US-02 | Scenario 1 | BDD | 11.2 |
| FR-014 | User guide generation | US-01 | Scenario 1 | BDD | 11.2 |
| FR-015 | Developer guide generation | US-01 | Scenario 1 | BDD | 11.2 |
| FR-016 | Changelog entries | US-01 | BDD Scenario 1 | BDD | 11.2 |
| FR-017 | Changelog prepend ordering | US-01 | BDD Scenario 1 | BDD | 11.2 |
| FR-018 | Inline code docs | US-01 | Scenario 1 | BDD | 11.2 |
| FR-019 | No logic modification constraint | US-01 | Scenario 1 | BDD | 11.2 |
| FR-020 | Docstring convention adherence | US-01 | Scenario 1 | BDD | 11.2 |

---

## 17. Technical References

- Documentation as Code, https://www.writethedocs.org/guide/docs-as-code/, consulted 2026-04-05

---

## 18. Version History

| Version | Date | Author | Summary of Changes |
|---------|------|--------|--------------------|
| 1.0 | 2026-04-05 | Spec Architect | Initial specification |
| 1.1 | 2026-04-05 | Spec Architect | Validation: added error behaviors, completed traceability matrix, removed ambiguous terms |
