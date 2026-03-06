---
name: step-05-cross-validation
description: "Check for contradictions, duplicate requirements, terminology consistency across story artifacts"
nextStepFile: './step-06-corrective-loop.md'
phase: sequential
phase_number: 3
executor: sub-agent
quality_gate: cross-validation
max_validator_retries: 2
skipToStepFile: './step-08-summary-report.md'
skipCondition: "all stories coherent — no corrections needed"
---

# Step 5: Cross-Validation

**Progress: Step 5 of 8** — Next: Corrective Loop (if issues) or Summary Report (if clean)
**Phase:** 3 (SEQUENTIAL — Cross-Story Coherence Check)

---

## STEP GOAL

Validate that all processed story artifacts are coherent as a set: no contradictions between stories, no duplicate requirements, consistent terminology, dependencies acknowledged, and epic goals fully covered. This is a holistic check that individual document processors cannot perform.

---

## MANDATORY EXECUTION RULES

- 🛑 This step examines ALL processed story artifacts together — not individually
- 📖 The cross-validator needs access to ALL artifact files and original stories
- 🚫 Cross-validator is READ-ONLY — no modifications to any artifacts or stories
- 🎯 Dispatch as a single sub-agent with full context across all stories

---

## SUB-AGENT CONTRACT: CROSS-VALIDATOR

Dispatch via Task tool with this contract:

```markdown
## Cross-Validator Sub-Agent Contract

**Role:** Cross-Story Coherence Validator
**Epic:** {epic_path}
**Stories Under Validation:** {count} stories
**Mode:** Intellectual — document artifacts only

### Artifacts to Examine (READ-ONLY)

{For each processed story:}
- **{story-id}**: 
  Original story: {story_file_path}
  Artifacts: {output_path}
  Artifact files:
    {list of artifact files with brief descriptions}
  Processing summary: {summary from processing_report}

### CRITICAL RESTRICTIONS
- ✅ You MAY: read any artifact file, story file, or epic file
- ✅ You MAY: compare content across artifacts
- 🛑 You MUST NOT: modify any files — artifacts, stories, or anything else
- 🛑 You MUST NOT: perform any git operations
- 🛑 You MUST NOT: create new files (your output is the report below)

### Your Mission

Examine all processed story artifacts as a COHESIVE SET and validate:

1. **No Contradictions Between Stories**: Story A's artifact doesn't contradict Story B's
   - Check business rules, data models, process flows
   - Compare stated assumptions across stories
   - Verify decision points are consistent

2. **No Duplicate Requirements Across Stories**: Different stories haven't defined the same requirement
   - Check for overlapping acceptance criteria
   - Look for redundant functionality across artifacts
   - Identify wasted effort or potential confusion

3. **Shared Terminology Is Consistent**: Same terms mean the same thing across all artifacts
   - Build a terminology inventory from all artifacts
   - Flag any term used with different meanings
   - Verify domain language is consistent

4. **Dependencies Between Stories Are Acknowledged**: If Story B depends on Story A, the artifacts reflect this
   - Check that referenced concepts from other stories exist
   - Verify data flow assumptions are consistent
   - Confirm interface contracts align

5. **Epic Goals Are Fully Covered**: The combined artifacts address all epic requirements
   - Re-read the epic file
   - Map each epic goal to story artifacts
   - Identify any gaps in coverage

### How to Examine

For each pair of stories:
1. Read both artifacts completely
2. Compare terminology, definitions, and assumptions
3. Check for contradictory statements
4. Verify cross-references are accurate

For epic coverage:
1. Extract all goals/requirements from the epic
2. For each goal, identify which artifact(s) address it
3. Flag any goals not covered by any artifact

### Reporting Format

```yaml
cross_validation_report:
  status: "coherent"  # or "issues_found"
  stories_validated: list[string]
  conflicts:
    - story_a: "{story-id-a}"
      story_b: "{story-id-b}"
      description: "Story A says user registration requires email; Story B says username-only"
      severity: "critical"  # critical | major | minor
      artifacts_involved:
        - path: "{artifact_path_a}"
          relevant_section: "Section heading or line reference"
        - path: "{artifact_path_b}"
          relevant_section: "Section heading or line reference"
      recommendation: "Align on email requirement — update Story B's artifact"
  gaps:
    - description: "Epic requires offline support but no story addresses it"
      affected_stories: list[string]
      severity: "major"
      recommendation: "Add a new story for offline support or clarify epic scope"
  duplicate_requirements:
    - description: "Both stories define user input validation rules independently"
      stories: list[string]
      recommendation: "Consolidate into a single shared validation spec"
  terminology_inconsistencies:
    - term: "user profile"
      usages:
        - story_id: "{story-id-a}"
          meaning: "Basic user info (name, email)"
        - story_id: "{story-id-b}"
          meaning: "Extended user data including preferences"
      recommendation: "Define 'user profile' in a shared glossary"
  dependency_issues:
    - description: "Story B references 'auth token format' from Story A but format differs"
      stories: list[string]
      recommendation: "Align token format specification"
  recommendations:
    - "Recommendation 1"
    - "Recommendation 2"
  summary: "Overall coherence assessment"
```
```

---

## EXECUTION SEQUENCE

### 0. Handle Zero Stories Case

IF `{stories_for_validation}` is empty (all stories failed or were deferred):

```
ℹ️ No stories available for cross-validation.
   All stories either failed processing or were deferred.
   Skipping validation — Loading Step 8: Summary Report...
```

Skip directly to `{skipToStepFile}` (step-08-summary-report.md).
Store default state:
```yaml
cross_validation_state:
  report: null
  status: "skipped_no_stories"
  critical_issues: []
  user_decision: "not_applicable"
  issues_for_correction: []
  validation_skipped: true
```

### 1. Handle Single Story Case (N=1)

IF only 1 story in `{stories_for_validation}`:

Pairwise cross-story checks (contradictions, duplicates, terminology) are vacuous with a
single story. Reduce the cross-validator scope to **epic coverage check only**:
- Does this single story's artifacts fully address the epic goals?
- Are there epic requirements left uncovered?

Modify the cross-validator contract to skip pairwise checks and focus on epic coverage.

### 2. Prepare Cross-Validation Context

Gather all information the cross-validator needs:

```yaml
validation_context:
  epic_file: "{epic_path}"
  epic_context: "{epic_context from step 1}"
  stories:
    - id: "{story-id-1}"
      story_file: "{path}"
      output_path: "{output_folder}/{story-id-1}/"
      artifacts: list[{path, description}]  # from collection step
      ac_coverage: "{percentage}"
      processing_summary: "{from step 4}"
    - id: "{story-id-2}"
      # ...
```

### 3. Dispatch Cross-Validator

Single Task tool invocation with the complete contract above.

**All placeholders resolved** — the cross-validator receives concrete file paths and summaries.

### 4. Handle Validator Failure

IF the cross-validator sub-agent fails (no output, error, or timeout):

```
⚠️ Cross-validator failed
Error: {error details or "no output received"}

Options:
[R] Retry validation (dispatch new cross-validator)
[S] Skip validation — proceed without cross-check (at your risk)
[M] Manual validation — I'll review coherence myself
```

- **R**: Re-dispatch cross-validator (max `{max_validator_retries}` retries — see workflow config)
- **S**: Skip validation, proceed to summary report with flag `validation_skipped: true`
- **M**: User validates manually, then tells orchestrator coherent/issues_found

### 5. Process Validation Report

Parse the `cross_validation_report` and classify issues:

| Issue Type | Severity | Action |
|-----------|----------|--------|
| Conflicts (critical) | 🔴 | Must resolve before completing |
| Conflicts (major) | 🟡 | Should resolve, user decides |
| Gaps (major) | 🟡 | May need additional stories |
| Duplicate requirements | 🟡 | User decides which to keep |
| Terminology inconsistencies | 🟡 | Should standardize |
| Dependency issues | 🔴 | Must resolve — artifacts reference incorrect data |

### 6. Present Cross-Validation Results

```
🔀 CROSS-VALIDATION RESULTS
═══════════════════════════════════════

Overall: {status}

{if conflicts exist}
🔴 CONTRADICTIONS DETECTED:
  • {story-a} ↔ {story-b}: {description}
    Artifacts: {artifact paths}
    Recommendation: {recommendation}
{/if}

{if gaps exist}
🟡 COVERAGE GAPS:
  • {gap_description}
    Affected: {stories}
    Recommendation: {recommendation}
{/if}

{if duplicate_requirements exist}
🟡 DUPLICATE REQUIREMENTS:
  • {description}
    Stories: {stories}
    Recommendation: {recommendation}
{/if}

{if terminology_inconsistencies exist}
🟡 TERMINOLOGY INCONSISTENCIES:
  • "{term}" used differently in {story_a} vs {story_b}
    Recommendation: {recommendation}
{/if}

{if dependency_issues exist}
🔴 DEPENDENCY ISSUES:
  • {description}
    Stories: {stories}
    Recommendation: {recommendation}
{/if}

{if no issues}
✅ All stories are coherent — no contradictions, gaps, or inconsistencies detected.
{/if}
```

### 7. Route Based on Results

**IF status == "coherent" (no issues found):**

```
✅ Cross-validation passed — all stories are coherent.
   Skipping corrective loop — Loading Step 8: Summary Report...
```

Load `{skipToStepFile}` (step-08-summary-report.md).

**IF status == "issues_found":**

Present options for critical/major issues:

```
⚠️ Cross-validation found issues requiring attention.

Options:
[C] Enter corrective loop — re-process affected stories with feedback
[M] Manual resolution — I'll fix the artifacts myself
[F] Force proceed — accept current state with known issues
[X] Abort
```

- **C**: Proceed to corrective loop (step 6) with validation feedback
- **M**: User fixes artifacts manually, then re-run cross-validation
- **F**: Proceed to summary with flag `issues_force_accepted: true`
- **X**: Abort workflow

### 8. Store Cross-Validation State

```yaml
cross_validation_state:
  report: {complete cross_validation_report}
  status: "coherent"  # or "issues_found"
  critical_issues: list[{description, stories}]
  user_decision: "corrective_loop"  # or "manual" or "force_proceed" or "abort"
  issues_for_correction: list[{story_id, feedback}]  # Stories to re-process
  validation_skipped: false
```

### 9. Proceed to Next Step

**IF entering corrective loop:**
```
⚠️ {issue_count} issues found across {story_count} stories.
   Loading Step 6: Corrective Loop...
```
Load, read completely, then execute `{nextStepFile}`.

**IF skipping to summary:**
```
✅ Cross-validation complete — no issues.
   Loading Step 8: Summary Report...
```
Load, read completely, then execute `{skipToStepFile}`.

---

## SUCCESS METRICS

- ✅ All story artifact pairs examined for contradictions
- ✅ Terminology inventory built and checked across all artifacts
- ✅ Epic coverage verified — all goals mapped to artifacts
- ✅ Dependency references validated across stories
- ✅ Critical issues escalated to user with clear options

## FAILURE MODES

- ❌ Only comparing adjacent stories (must check ALL pairs)
- ❌ Not checking terminology consistency
- ❌ Not verifying epic coverage
- ❌ Silently proceeding past critical contradictions
- ❌ Cross-validator modifying artifact files
