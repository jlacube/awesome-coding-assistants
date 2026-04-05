---
skill: review-security
wp: WP11-data-model-api-design-skills
spec: .sdd/specs/002-spec-architect-v2.spec.md
reviewed_at: 2026-04-05T15:05:00Z
status: completed
finding_counts:
  pass: 2
  warn: 0
  fail: 0
  na: 12
files_reviewed:
  - .github/skills/spec-data-model/SKILL.md
  - .github/skills/spec-api-design/SKILL.md
---

# review-security Findings for WP11-data-model-api-design-skills

## Summary

Evaluated 14 OWASP Secure Coding Practice categories against the two SKILL.md files. Both files are markdown instruction documents, not executable code -- they do not handle user input, manage sessions, perform authentication, access databases, or execute system commands. 12 of 14 OWASP categories are N/A. Two categories produce PASS findings based on the security-relevant instructions embedded in the skills.

## Findings

### SEC-001 [PASS]
- **Checklist item**: OWASP 1 - Input Validation
- **Requirement**: Skills instruct the LLM to validate inputs
- **File**: .github/skills/spec-api-design/SKILL.md#L85-L90
- **Description**: The spec-api-design skill instructs that every endpoint request body SHALL have "typed input schema with field-level validation rules." This correctly propagates input validation into generated specifications, ensuring downstream code will have validation requirements.

### SEC-002 [PASS]
- **Checklist item**: OWASP 5 - Authorization
- **Requirement**: Skills instruct auth requirements
- **File**: .github/skills/spec-api-design/SKILL.md#L72, .github/skills/spec-api-design/SKILL.md#L88
- **Description**: The spec-api-design skill requires every endpoint to declare auth requirements explicitly and lists 401/403 as mandatory error codes for authenticated endpoints. This ensures generated specs include authorization requirements.

### SEC-003 [N/A]
- **Checklist item**: OWASP 2 - Output Encoding
- **Justification**: SKILL.md files are markdown instructions consumed by an LLM. They do not produce output rendered in browsers or other contexts requiring encoding.

### SEC-004 [N/A]
- **Checklist item**: OWASP 3 - Authentication
- **Justification**: SKILL.md files do not implement authentication. They instruct the LLM to document auth requirements in specs.

### SEC-005 [N/A]
- **Checklist item**: OWASP 4 - Session Management
- **Justification**: SKILL.md files do not manage sessions.

### SEC-006 [N/A]
- **Checklist item**: OWASP 6 - Cryptographic Practices
- **Justification**: SKILL.md files do not perform cryptographic operations.

### SEC-007 [N/A]
- **Checklist item**: OWASP 7 - Error Handling and Logging
- **Justification**: SKILL.md files are instructions, not executable code with error handling.

### SEC-008 [N/A]
- **Checklist item**: OWASP 8 - Data Protection
- **Justification**: SKILL.md files do not store or transmit sensitive data.

### SEC-009 [N/A]
- **Checklist item**: OWASP 9 - Communication Security
- **Justification**: SKILL.md files do not communicate over networks.

### SEC-010 [N/A]
- **Checklist item**: OWASP 10 - System Configuration
- **Justification**: SKILL.md files do not configure systems.

### SEC-011 [N/A]
- **Checklist item**: OWASP 11 - Database Security
- **Justification**: SKILL.md files do not interact with databases.

### SEC-012 [N/A]
- **Checklist item**: OWASP 12 - File Management
- **Justification**: SKILL.md files are static markdown. File operations occur at LLM runtime, not within the skill file itself.

### SEC-013 [N/A]
- **Checklist item**: OWASP 13 - Memory Management
- **Justification**: SKILL.md files are not executable code.

### SEC-014 [N/A]
- **Checklist item**: OWASP 14 - General Coding Practices
- **Justification**: SKILL.md files are markdown instructions, not compiled/interpreted code.
