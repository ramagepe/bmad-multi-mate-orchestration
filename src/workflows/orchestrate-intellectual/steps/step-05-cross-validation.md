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
  Story file: {stories_output_path}/{story_key}.md
  Processing summary: {summary from processing_report}

### CRITICAL RESTRICTIONS
- ✅ You MAY: read any artifact file, story file, or epic file
- ✅ You MAY: compare content across artifacts
- 🛑 You MUST NOT: modify any files — artifacts, stories, or anything else
- 🛑 You MUST NOT: perform any git operations
- 🛑 You MUST NOT: create new files (your output is the report below)
- 🛑 SUBAGENT-STOP: You are a SUB-AGENT dispatched for a specific task.
  Do NOT invoke BMO orchestration workflows or BMAD agent menus.
  Do NOT dispatch your own sub-agents via Task tool.
  Complete YOUR assigned task and report back. Nothing else.

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

6. **File Ownership Clarity**: For shared files modified by multiple stories, verify ownership is unambiguous
   - Each shared file should have exactly ONE story that CREATEs it
   - Subsequent stories must use VERIFY/MODIFY/ADD — never a second CREATE for the same file
   - Each VERIFY/MODIFY/ADD should reference the owning story (e.g., `[Owner: Story 1.1]`)
   - Check the File List table in each story for correct action verbs
   - Flag files where ownership is ambiguous, contested, or where multiple stories CREATE the same file

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
  file_ownership_issues:
    - file: "path/to/shared/file.ts"
      description: "Multiple stories CREATE this file instead of one CREATE + others VERIFY/MODIFY"
      stories: list[string]
      canonical_owner: "{story-id that should CREATE}"
      recommendation: "Story X CREATEs, others use VERIFY/MODIFY with [Owner: Story X]"
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
  stories_output_path: "{stories_output_path}"  # Directory containing all story files
  stories:
    - id: "{story-id-1}"
      story_file: "{stories_output_path}/{story-key-1}.md"
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

### 5. Persist Full Cross-Validation Analysis

🛑 **IMMEDIATELY after receiving the cross-validator's output**, write the COMPLETE analysis to disk — not just the YAML summary. The sub-agent's full output includes narrative analysis, file-by-file comparisons, code snippet comparisons, and reasoning that explains WHY each issue exists. This is critical audit trail that does not survive the session otherwise.

**Write to:** `{output_folder}/_orchestration/cross-validation-report.md`

**Content structure:**
```markdown
# Cross-Validation Report — {epic_name}
Generated: {timestamp}
Stories validated: {count}

## Analysis

{FULL narrative analysis from the cross-validator sub-agent — every section,
every file comparison, every code snippet, every observation. Do NOT summarize
or truncate. Copy the sub-agent's complete analysis output verbatim.}

## Structured Report

{The YAML cross_validation_report block from the sub-agent}
```

This file may be large. That is intentional — it serves as the permanent record of cross-validation reasoning. Step-08's summary report provides the condensed view.

### 6. Process Validation Report

Parse the `cross_validation_report` and classify each issue into one of two categories:

**AUTO-RESOLVABLE** — The cross-validator provided a clear, unambiguous recommendation AND:
- The fix is a content correction (terminology, syntax, conventions, file ownership)
- The authoritative source is clear (architecture.md, epic conventions, project config)
- No scope decisions or trade-offs are involved

**NEEDS-HUMAN** — The issue requires human judgment because:
- The cross-validator's recommendation is ambiguous or says "decide between X and Y"
- The fix involves a scope decision (add/remove functionality)
- Two authoritative sources contradict each other
- The issue is a coverage gap requiring a new story

| Issue Type | Severity | Auto-Resolvable? |
|-----------|----------|-------------------|
| Conflicts with clear fix (e.g., wrong syntax per architecture) | 🔴/🟡 | ✅ YES — auto-correct with source citation |
| Conflicts requiring scope decision | 🔴/🟡 | ❌ NO — needs human |
| Duplicate requirements with clear owner | 🟡 | ✅ YES — assign to canonical owner per dependency chain |
| Terminology inconsistencies | 🟡 | ✅ YES — standardize per architecture/epic terminology |
| Dependency issues with clear fix | 🔴 | ✅ YES — align per architecture contracts |
| Coverage gaps requiring new stories | 🟡 | ❌ NO — needs human (scope change) |

### 7. Present Cross-Validation Results

```
🔀 CROSS-VALIDATION RESULTS
═══════════════════════════════════════

Overall: {status}

{For each issue, grouped by severity:}
🔴/🟡 [{AUTO|HUMAN}] {description}
  Stories: {story_a}, {story_b}
  Recommendation: {recommendation}
  {if AUTO} Source: {authoritative source for the fix}
  {if HUMAN} Reason: {why this needs human input}

{if no issues}
✅ All stories are coherent — no contradictions, gaps, or inconsistencies detected.
{/if}
```

### 8. Route Based on Results

**IF status == "coherent" (no issues found):**

```
✅ Cross-validation passed — all stories are coherent.
   Skipping corrective loop — Loading Step 8: Summary Report...
```

Load `{skipToStepFile}` (step-08-summary-report.md).

**IF status == "issues_found" and ALL issues are AUTO-RESOLVABLE:**

Do NOT ask the user. Proceed directly to corrective loop:

```
⚠️ Cross-validation found {issue_count} issues — all auto-resolvable.
   Entering corrective loop automatically with cross-validator recommendations...
   Loading Step 6: Corrective Loop...
```

Load `{nextStepFile}` with all issues queued for correction.

**IF status == "issues_found" and SOME issues are NEEDS-HUMAN:**

Auto-enter corrective loop for the auto-resolvable issues, then present ONLY the human-required issues:

```
⚠️ Cross-validation found {total_count} issues.
   🤖 {auto_count} auto-resolvable — entering corrective loop automatically.
   🧑 {human_count} need your input:

{For each NEEDS-HUMAN issue:}
  [{number}] [{severity}] {description}
      Stories: {stories}
      Options: {specific options for this issue}

How do you want to handle the human-required issues?
[R] Resolve now — I'll answer each one, then corrective loop handles everything
[D] Defer — run corrective loop for auto-resolvable issues first, ask me after
[F] Force proceed — accept all current state with known issues
[X] Abort
```

- **R**: User provides decisions for each human issue, ALL issues enter corrective loop together
- **D**: Corrective loop runs for auto-resolvable issues first. After loop completes, human issues are re-presented. If cross-validation shows they're resolved as a side-effect of other corrections → skip. If still open → present again.
- **F**: Proceed to summary with flag `issues_force_accepted: true`
- **X**: Abort workflow

### 9. Store Cross-Validation State

```yaml
cross_validation_state:
  report: {complete cross_validation_report}
  status: "coherent"  # or "issues_found"
  auto_resolvable_issues: list[{description, stories, recommendation, source}]
  human_required_issues: list[{description, stories, options, reason}]
  user_decision: "auto_corrective"  # or "resolve_now" or "defer" or "force_proceed" or "abort"
  issues_for_correction: list[{story_id, feedback, resolution_source}]
  validation_skipped: false
```

### 10. Proceed to Next Step

**IF entering corrective loop (auto or after user decisions):**
```
⚠️ {issue_count} issues queued for correction ({auto_count} auto + {human_resolved_count} human-decided).
   Loading Step 6: Corrective Loop...
```
Load, read completely, then execute `{nextStepFile}`.

**IF skipping to summary (coherent or force-proceed):**
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
