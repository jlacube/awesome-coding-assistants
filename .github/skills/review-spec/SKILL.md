---
name: review-spec
description: "Spec adherence review skill. Evaluates implementation against functional requirements, success criteria, and acceptance scenarios from the specification. Verifies SHALL obligations, preconditions, postconditions, error paths, edge cases, data models, and API contracts."
argument-hint: "Invoked by Review Coordinator - do not call directly"
---

# review-spec - Spec Adherence Review Skill

This skill is invoked by the Review Coordinator as a subagent. It receives a WP identifier and spec path, discovers and reads all implementation code relevant to the WP, evaluates each functional requirement against the code, and writes structured findings to the specified output path.

**Input contract** (received via subagent prompt):
1. Read this SKILL.md file for review instructions.
2. Read the specification file to extract functional requirements.
3. Discover and read all implementation code relevant to this WP.
4. Evaluate each checklist item below against the discovered code.
5. Write structured findings to the specified output path.
6. Return a brief summary (counts of PASS/WARN/FAIL/N/A).

**Constraint**: Do NOT modify any source code, the WP file, or the spec file. Only write to the specified output path.

---

## 1. Identify In-Scope FRs

1. Read the WP file and locate the `Spec References` section.
2. For each spec section referenced, identify the functional requirements (FR-XXX) that fall within those sections.
3. Also read the WP's task descriptions to identify any additional FRs mentioned.
4. Build a list of all FRs that this WP is responsible for implementing.

---

## 2. FR Classification Checklist

For each in-scope FR, classify implementation adherence as one of:

| Classification | Definition | Severity |
|---------------|------------|----------|
| **Compliant** | The FR is fully implemented exactly as specified. All obligations, preconditions, postconditions, and error paths are satisfied. | PASS |
| **Partial** | Some aspects of the FR are implemented but others are missing or incomplete. | FAIL |
| **Deviating** | The FR is implemented but behaves differently from the specification. | FAIL |
| **Missing** | The FR is not implemented at all. | FAIL |

### Detailed verification per FR

For each FR, check all of the following:

1. **SHALL/SHALL NOT obligation**: Is the exact obligation stated in the FR satisfied? Look for the specific behavior described.
2. **Preconditions enforced**: Are the preconditions listed in the FR checked before the main logic executes?
3. **Postconditions produced**: Does the code produce the outputs/state changes described as postconditions?
4. **Error paths handled**: Are all error conditions listed in the FR (under "Error:" bullets) handled with the specified behavior?
5. **Edge cases covered**: Are edge cases from the spec's acceptance scenarios (Section 5, Section 11.2) addressed in the implementation?
6. **Data model match**: Do data model fields, types, and validation rules match the spec's Section 7?
7. **API contract match**: Do API request/response schemas match the spec's Section 8?
8. **Error codes match**: Do returned error codes match the spec's error taxonomy?

For each checklist item that fails, cite the specific evidence: the code that deviates, the spec text that defines the expected behavior, and the file/line where the deviation occurs.

---

## 3. Stub Detection

The following patterns indicate stub implementations. Classify these as **Missing** (not Partial):

- `pass` as the sole statement in a function body (Python)
- `raise NotImplementedError` or `raise NotImplementedError(...)` (Python)
- `...` (ellipsis) as function body (Python)
- `# TODO`, `# FIXME`, `# HACK` as the only meaningful content in a function
- Empty function bodies: `{}` with no logic (JavaScript/TypeScript)
- `throw new Error("Not implemented")` or similar placeholder throws
- Any function body with fewer than 3 meaningful lines (excluding comments and whitespace) that does not perform the FR's specified behavior

When a stub is detected:
- Classification: **Missing**
- Finding severity: **FAIL**
- Evidence: show the stub code and the FR it should implement

---

## 4. Success Criteria Verification

For each success criterion (SC-XXX) referenced by the WP:

1. **Locate evidence**: Look for passing tests, observable behavior, or measurable metrics that demonstrate the SC is met.
2. **Verify evidence is genuine**: Confirm the claimed test file actually exists and its assertions are not vacuous (no `assert True`, no empty test bodies, no fully mocked subjects).
3. **Handle unverifiable SCs**: If an SC cannot be verified at this stage (e.g., requires runtime behavior not yet available), document it as N/A with justification: "Deferred verification: <reason>".

| Evidence status | Severity |
|----------------|----------|
| Genuine evidence exists | PASS |
| Evidence is fabricated (test does not exist or is vacuous) | FAIL |
| No evidence provided for a verifiable SC | FAIL |
| SC cannot be verified at this stage | N/A (with justification) |

---

## 5. Severity Rules

| Finding type | Severity | Notes |
|-------------|----------|-------|
| FR classified as Compliant | PASS | All 8 checklist items verified |
| FR classified as Partial | FAIL | List which items are missing |
| FR classified as Deviating | FAIL | Describe the deviation |
| FR classified as Missing | FAIL | Note if stub detected |
| SC-XXX with genuine evidence | PASS | |
| SC-XXX with missing/fabricated evidence | FAIL | |
| SC-XXX not verifiable at this stage | N/A | Include justification |
| FR not applicable to this WP | N/A | Include justification |

**There is no WARN level for spec adherence.** Spec compliance is binary: either the requirement is fully met or it is not.

Every N/A finding MUST include a justification field explaining why the item does not apply. Do NOT produce findings for checklist items that are not applicable -- record them as N/A with justification instead.

---

## 6. Output Format

Write findings to the specified output path using this exact format:

### YAML Frontmatter

```yaml
---
skill: review-spec
wp: <WP-id>
spec: <spec_path>
reviewed_at: <ISO 8601 timestamp>
status: completed
finding_counts:
  pass: <count>
  warn: 0
  fail: <count>
  na: <count>
files_reviewed:
  - <file1>
  - <file2>
---
```

### Findings Body

```markdown
# review-spec Findings for <WP-id>

## Summary

<Brief overview of scope and overall assessment. List total FRs evaluated, how many Compliant vs Partial/Deviating/Missing.>

## Findings

### SPEC-001 [PASS]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-001
- **File**: <file_path>
- **Description**: <What was verified and found correct>

### SPEC-002 [FAIL]
- **Checklist item**: FR classification - SHALL obligation
- **Requirement**: FR-012
- **File**: <file_path>#L<start>-L<end>
- **Description**: <What was found wrong or missing>
- **Expected**: <What the spec requires>
- **Evidence**:
  ```
  <code snippet showing the deviation>
  ```

### SPEC-003 [N/A]
- **Checklist item**: API contract match
- **Justification**: No API endpoints in this WP - all artifacts are markdown files.
```

### Rules

- Finding IDs use prefix `SPEC-` and are sequential: SPEC-001, SPEC-002, etc. No gaps.
- Every FAIL finding MUST include: Checklist item, Requirement, File (with line range), Description, Expected, Evidence.
- Every PASS finding MUST include: Checklist item, Requirement, File, Description.
- Every N/A finding MUST include: Checklist item, Justification.
- `finding_counts` MUST accurately reflect the actual findings in the file.
- `files_reviewed` MUST list every file that was read and evaluated during this review.
- `warn` count is always 0 for this skill (spec adherence has no WARN level).
