---
name: step-06-corrective-loop
description: "Re-dispatch document processors with cross-validation feedback, with circuit breaker"
nextStepFile: './step-07-human-escalation.md'
phase: corrective
phase_number: 3
executor: mixed
max_correction_loops: "{max_correction_loops}"
skipToStepFile: './step-08-summary-report.md'
skipCondition: "all issues resolved — no escalation needed"
---

# Step 6: Corrective Loop

**Progress: Step 6 of 8** — Next: Human Escalation (if circuit breaker) or Summary Report (if resolved)
**Phase:** 3 (CORRECTIVE — Process + Validate cycles)
**Circuit Breaker:** Max `{max_correction_loops}` iterations per story (default: 3)

---

## STEP GOAL

For each story with issues found by cross-validation: re-dispatch the document processor with specific feedback, then re-validate. Repeat until coherent or circuit breaker triggers. This is the quality convergence mechanism for intellectual mode.

---

## MANDATORY EXECUTION RULES

- 🛑 **CIRCUIT BREAKER**: Max `{max_correction_loops}` attempts per story — then ESCALATE to human (step 7)
- 📖 Each correction attempt includes the FULL cross-validation feedback
- 🚫 NEVER exceed the circuit breaker limit silently — always escalate to user
- 🎯 Track loop count per story independently
- 📋 After each correction, re-run cross-validation to check if issues are resolved

---

## CORRECTIVE LOOP ALGORITHM

```
FOR each story in issues_for_correction:
  loop_count = 0

  WHILE has_issues AND loop_count < max_correction_loops:
    loop_count += 1

    1. Dispatch document processor with cross-validation feedback
    2. Processor revises artifacts
    3. Verify revised artifacts exist
    4. Re-dispatch cross-validator (targeted — only affected stories)
    5. IF coherent → EXIT loop (story resolved)
    6. IF issues_found → CONTINUE loop
    7. IF processor failed → ESCALATE immediately

  IF loop_count >= max_correction_loops AND has_issues:
    → Add to escalation queue (step 7)
```

---

## EXECUTION SEQUENCE

### 0. Handle Skip Case

IF this step was reached but no stories need correction (edge case):

Default state for downstream steps:
```yaml
corrective_state:
  correction_tracking: []
  total_correction_loops: 0
  stories_resolved: "{from cross_validation_state.stories — all}"
  stories_for_escalation: []
  stories_dropped: []
```

Skip to summary report: Load `{skipToStepFile}`.

### 1. Initialize Loop Tracking

```yaml
correction_tracking:
  - story_id: "{story-id}"
    loop_count: 0
    max_loops: "{max_correction_loops}"
    status: "needs_correction"
    issues:
      - type: "{conflict | gap | terminology | dependency}"
        description: "{from cross-validation report}"
        related_stories: list[string]
    history:
      - iteration: 0
        validation_findings: {original cross-validation findings for this story}
```

### 2. Dispatch Corrective Document Processor

For each story needing correction, dispatch via Task tool with ENHANCED contract:

```markdown
## Corrective Document Processor Contract

**Role:** Document Processor — CORRECTION MODE
**Story file to correct:** {stories_output_path}/{story_key}.md
**Correction Iteration:** {loop_count} of {max_correction_loops}

### Context: You are FIXING issues found in cross-validation

Your previously created story file was validated against other stories in the same epic
and the following issues were found. You MUST revise the story file to resolve
these issues while maintaining correctness of your original work.

### Cross-Validation Feedback to Address:

{For each issue affecting this story:}
**[{severity}] {type}**: {description}
  Resolution type: {AUTO-RESOLVED | HUMAN-DECIDED}
  Related stories: {related_story_ids}
  Your story file: {stories_output_path}/{story_key}.md
  Relevant section: {section reference}
  Recommendation: {recommendation from cross-validator}
  Authoritative source: {e.g., "architecture.md § Module System" or "User decision: chose option A"}
  
> ⚠️ You MUST apply the recommendation as stated — it was validated against the authoritative source. Do NOT re-interpret or substitute your own judgment. If the recommendation is impossible to apply, report it as a `remaining_concern` instead of silently ignoring it.

### Other Story Context (for alignment)

{For each related story — provide READ-ONLY context:}
- **{related-story-id}**: 
  Story file: {stories_output_path}/{related_story_key}.md
  Summary: {story summary}
  Key terms: {relevant terminology from that story}

### Your Output
Modify the story file at `{stories_output_path}/{story_key}.md` in-place.
This is the same file you created — revise it directly.

### CRITICAL RESTRICTIONS (same as original)
- ✅ You MAY: read any file in the project for context
- ✅ You MAY: read OTHER stories' files for alignment (READ-ONLY)
- ✅ You MAY: modify the story file at {stories_output_path}/{story_key}.md
- 🛑 You MUST NOT: modify other stories' files
- 🛑 You MUST NOT: modify files outside {stories_output_path}/
- 🛑 You MUST NOT: perform any git operations

### Exit Criteria for Correction
- ALL cross-validation issues addressed in revised artifacts
- Terminology aligned with related stories
- Dependencies and references updated to be consistent
- **CRITICAL: Original acceptance criteria coverage must NOT regress** — do not remove
  or weaken content that addresses ACs in order to fix cross-validation issues. If a
  conflict exists between an AC and cross-story coherence, flag it as a remaining concern
  rather than sacrificing AC coverage

### Reporting Format

```yaml
correction_report:
  story_id: "{story-id}"
  iteration: {loop_count}
  status: "completed"  # or "failed" or "blocked"
  issues_addressed:
    - issue: "{description from feedback}"
      resolution: "How it was resolved in the artifact"
      resolution_type: "auto"  # or "human-decided"
      authoritative_source: "architecture.md § Module System"  # What justified the decision
      artifact_modified: "{file path}"
  artifacts_modified: list[string]
  artifacts_created: list[string]
  summary: "What was corrected and how"
  remaining_concerns: list[string]
```
```

### 3. Verify Corrected Story File

After processor completes, verify the revised story file exists and was actually modified:

```bash
# Check story file exists and get size/mtime
ls -la {stories_output_path}/{story_key}.md

# Check file is not empty
test -s {stories_output_path}/{story_key}.md
```

**Verification criteria:**
- Story file exists and is non-empty
- File modification time is newer than pre-correction timestamp
- File size is reasonable (not truncated)

### 4. Re-Dispatch Targeted Cross-Validator

Send a TARGETED cross-validation — only checking the corrected story against its related stories:

```markdown
### Additional Context: This is correction iteration {loop_count}

Previous issues that were addressed:
{list of issues from previous validation}

Revised artifacts:
{list of modified artifacts}

Please verify:
1. The specific issues have been properly resolved
2. No NEW issues introduced by the corrections
3. Terminology is now consistent with related stories
4. Dependencies are properly acknowledged
```

**Optimization:** Instead of re-validating ALL stories, only validate the corrected story against the stories it had conflicts/issues with. This reduces validator scope and speeds up the loop.

### 5. Evaluate Re-Validation Result

```
IF validation_status == "coherent" (for this story):
  → Move story to stories_resolved
  → Log: "Story {id} resolved after {loop_count} correction(s)"
  → EXIT loop for this story

IF validation_status == "issues_found":
  → Increment loop_count
  → IF loop_count < max_correction_loops:
    → Record new findings in history
    → CONTINUE loop (go to step 2)
  → IF loop_count >= max_correction_loops:
    → Add to escalation queue (go to step 7)

IF processor_status == "failed":
  → ESCALATE immediately regardless of loop count
```

### 6. Present Corrective Loop Summary

After all stories have exited the loop (resolved, exhausted, or failed):

```
🔄 CORRECTIVE LOOP SUMMARY
═══════════════════════════════════════

Total correction cycles executed: {total_across_all_stories}

  ✅ {story-id-1}: Resolved after 1 correction
  ✅ {story-id-2}: Resolved after 2 corrections
  ⚠️ {story-id-3}: Circuit breaker — 3 corrections, still has issues
  ❌ {story-id-4}: Processor failed during correction

Stories resolved: {resolved_count}
Stories for escalation: {escalation_count}
```

### 7. Store Corrective State

```yaml
corrective_state:
  correction_tracking: {complete history per story}
  total_correction_loops: int
  stories_resolved: list[string]
  stories_for_escalation: list[{story_id, remaining_issues, loop_count}]
  stories_dropped: list[string]
  auto_resolved_decisions:  # For step-08 summary report — audit trail
    - issue: "{description}"
      resolution: "{what was done}"
      authoritative_source: "{source that justified the decision}"
      stories_affected: list[string]
      iteration_resolved: int
```

### 8. Route to Next Step

**IF stories need escalation (circuit breaker triggered):**
```
⚠️ {escalation_count} stories exhausted correction attempts.
   Loading Step 7: Human Escalation...
```
Load, read completely, then execute `{nextStepFile}`.

**IF all stories resolved (no escalation needed):**
```
✅ All corrective issues resolved!
   Loading Step 8: Summary Report...
```
Load, read completely, then execute `{skipToStepFile}`.

---

## SUCCESS METRICS

- ✅ Each story gets up to `{max_correction_loops}` attempts
- ✅ Full cross-validation feedback included in each correction dispatch
- ✅ Circuit breaker triggers at the limit — never silently exceeded
- ✅ Targeted re-validation after each correction (not full re-run)
- ✅ Complete correction history tracked per story

## FAILURE MODES

- ❌ Exceeding circuit breaker limit without escalation
- ❌ Not including cross-validation feedback in correction contract
- ❌ Not verifying corrected artifacts exist
- ❌ Running full cross-validation instead of targeted (wasteful)
- ❌ Losing correction history between iterations
- ❌ Auto-resolving without actual re-validation
