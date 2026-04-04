---
skill: review-security
wp: WP06-p2-skills
spec: .sdd/specs/001-reviewer-v2-skill-based-architecture.spec.md
reviewed_at: 2026-04-04T12:00:00Z
status: completed
finding_counts:
  pass: 3
  warn: 1
  fail: 1
  na: 14
files_reviewed:
  - .github/skills/review-tests/SKILL.md
  - .github/skills/review-architecture/SKILL.md
  - .sdd/plans/WP06-p2-skills.md
---

# review-security Findings for WP06-p2-skills

## Summary

Reviewed 2 SKILL.md instruction files (review-tests, review-architecture) delivered by WP06. These are markdown-based review checklists consumed by VS Code Copilot Chat subagents -- they are not executable code, do not handle user input, authentication, sessions, databases, cryptography, or network communication. As a result, 14 of the 14 OWASP Secure Coding Practices categories are either fully N/A or have no applicable checklist items beyond what is evaluated below.

One FAIL was identified: review-tests instructs the subagent to "run the coverage tool," which directly violates NFR-004 (static analysis only -- skills shall not execute discovered code). One WARN was identified: neither P2 skill includes an explicit constraint against reproducing secret values found in reviewed code (NFR-005 gap).

## Findings

### SEC-001 [FAIL]
- **Checklist item**: General Coding Practices - No dynamic code execution with user-supplied data
- **Requirement**: NFR-004 ("Skills SHALL NOT execute any discovered code. Review is static analysis only.")
- **File**: .github/skills/review-tests/SKILL.md#L51
- **Description**: The Coverage Thresholds dimension instructs the subagent to "run the coverage tool (e.g., `pytest --cov --cov-branch`) and report actual thresholds." Executing `pytest` runs the project's test suite and transitively the project's source code, violating the spec's static-analysis-only constraint.
- **Expected**: Coverage evaluation should be performed by static inspection of coverage configuration and test file coverage, or by reading existing coverage reports -- not by executing test runners. The instruction should be rewritten to avoid directing the subagent to execute code.
- **Evidence**:
  ```markdown
  **If coverage tooling is configured**: run the coverage tool (e.g., `pytest --cov --cov-branch`) and report actual thresholds.
  ```

### SEC-002 [WARN]
- **Checklist item**: Data Protection - No secrets in review artifacts
- **Requirement**: NFR-005 ("The coordinator SHALL NOT store credentials, tokens, or API keys in any review artifact. If a skill finds a hardcoded secret during review, it SHALL reference the file and line but SHALL NOT reproduce the secret value in the findings file.")
- **File**: .github/skills/review-tests/SKILL.md#L20
- **Description**: The constraint section states "Do NOT modify any test files, source code, WP file, or spec file" (FR-028) but does not include an NFR-005-equivalent instruction to avoid reproducing secret values in evidence code snippets. While review-tests is unlikely to encounter secrets in its normal review domain (test files), code excerpts included as evidence could inadvertently contain hardcoded credentials from test fixtures or configuration.
- **Expected**: Both P2 skill files should include a constraint line equivalent to the one in review-security: "Do NOT reproduce actual secret values (API keys, tokens, passwords) in findings -- cite file and line only."
- **Evidence**:
  ```markdown
  **Constraint**: Do NOT modify any test files, source code, WP file, or spec file. Only write to the specified output path (FR-028).
  ```
  Missing: no NFR-005 constraint. Compare with review-security SKILL.md which explicitly states: "Do NOT reproduce actual secret values (API keys, tokens, passwords) in findings - cite file and line only (NFR-005)."

### SEC-003 [PASS]
- **Checklist item**: Data Protection - No secrets as literals in source code
- **Requirement**: OWASP SCP Category 8
- **File**: .github/skills/review-tests/SKILL.md, .github/skills/review-architecture/SKILL.md
- **Description**: Neither SKILL.md file contains any hardcoded credentials, API keys, tokens, or secrets. All content is instructional markdown.

### SEC-004 [PASS]
- **Checklist item**: File Management - Write scope constrained
- **Requirement**: FR-028, OWASP SCP Category 12
- **File**: .github/skills/review-tests/SKILL.md#L20, .github/skills/review-architecture/SKILL.md#L20
- **Description**: Both skills include explicit constraints restricting write operations to the specified output path only. The read-only constraint (FR-028) is correctly stated in both files, preventing unintended file modification.

### SEC-005 [PASS]
- **Checklist item**: Communication Security - Web research URL restrictions
- **Requirement**: NFR-006
- **File**: .github/skills/review-tests/SKILL.md, .github/skills/review-architecture/SKILL.md
- **Description**: Neither P2 skill uses or references web research (`#tool:web`). No arbitrary URL fetching is instructed. NFR-006 compliance is satisfied by absence of web research usage.

### SEC-006 [N/A]
- **Checklist item**: Input Validation - All items
- **Justification**: Markdown instruction files. No user input processing, no server-side validation context. Skills are consumed as static text by a VS Code subagent.

### SEC-007 [N/A]
- **Checklist item**: Output Encoding - All items
- **Justification**: Markdown instruction files. No HTTP responses, no rendered output to clients, no encoding context.

### SEC-008 [N/A]
- **Checklist item**: Authentication and Password Management - All items
- **Justification**: No authentication system. Skills are static instruction files with no auth logic.

### SEC-009 [N/A]
- **Checklist item**: Session Management - All items
- **Justification**: No session management. Skills are stateless instruction files.

### SEC-010 [N/A]
- **Checklist item**: Access Control - All items
- **Justification**: No access control system. Skills are read by the VS Code subagent framework with no authorization layer.

### SEC-011 [N/A]
- **Checklist item**: Cryptographic Practices - All items
- **Justification**: No cryptographic operations. Skills are plain markdown files.

### SEC-012 [N/A]
- **Checklist item**: Error Handling and Logging - All items
- **Justification**: No runtime error handling or logging. Skills are markdown instruction files, not executable code. Error handling for skill dispatch failures is owned by the coordinator (FR-007), not the skills themselves.

### SEC-013 [N/A]
- **Checklist item**: Communication Security - TLS, certificates, connections
- **Justification**: No network communication. All operations are local file reads and writes within the VS Code workspace.

### SEC-014 [N/A]
- **Checklist item**: System Configuration - All items
- **Justification**: No system configuration. Skills are markdown files with no deployable components, HTTP methods, or security headers.

### SEC-015 [N/A]
- **Checklist item**: Database Security - All items
- **Justification**: No database access in this WP. Skills review code but do not interact with databases.

### SEC-016 [N/A]
- **Checklist item**: File Management - User-supplied data in paths, uploads, redirects
- **Justification**: Skills do not process user-supplied file paths, handle uploads, or perform redirects. File output path is provided by the coordinator via subagent prompt, not by end users.

### SEC-017 [N/A]
- **Checklist item**: Memory Management - All items
- **Justification**: Markdown instruction files. No executable code, no memory allocation, no buffer management. Managed by the VS Code runtime environment.

### SEC-018 [N/A]
- **Checklist item**: General Coding Practices - OS commands, checksums, locking, variable initialization
- **Justification**: No executable code beyond the coverage tool instruction (addressed in SEC-001). Remaining general coding practice items (OS commands from user input, checksums, locking, variable initialization) are not applicable to markdown instruction files.

### SEC-019 [N/A]
- **Checklist item**: Cryptographic Practices - FIPS 140-2 compliance
- **Justification**: No cryptographic operations. Not applicable to markdown instruction files.

## Spec Security Cross-Reference

### NFR-004: Static analysis only
- **Status**: FAIL (SEC-001)
- **Detail**: review-tests SKILL.md line 51 instructs subagent to execute `pytest --cov`, violating the static-analysis-only constraint.

### NFR-005: No secrets in review artifacts
- **Status**: WARN (SEC-002)
- **Detail**: Neither P2 skill explicitly constrains subagents from reproducing secret values in findings evidence. The P1 review-security skill includes this constraint; P2 skills do not.

### NFR-006: Web research URL restrictions
- **Status**: PASS (SEC-005)
- **Detail**: Neither P2 skill uses web research. No arbitrary URL fetching.
