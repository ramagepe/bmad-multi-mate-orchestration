---
name: step-07-corrective-loop
description: "Re-dispatch implementors with review feedback, with circuit breaker"
nextStepFile: './step-08-cross-validation.md'
phase: corrective
phase_number: 3
executor: mixed
max_correction_loops: "{max_correction_loops}"
---

# Step 7: Corrective Loop

**Progress: Step 7 of 14** — Next: Cross-Validation
**Phase:** 3 (CORRECTIVE — Implementor + Reviewer cycles)
**Circuit Breaker:** Max `{max_correction_loops}` iterations per story (default: 3)

---

## STEP GOAL

For each story with `needs_changes` status: re-dispatch the implementor with reviewer feedback, then re-review. Repeat until approved or circuit breaker triggers. This is the quality convergence mechanism.

---

## MANDATORY EXECUTION RULES

- 🛑 **CIRCUIT BREAKER**: Max `{max_correction_loops}` attempts per story — then ESCALATE
- 📖 Each correction attempt includes the FULL review feedback
- 🚫 NEVER exceed the circuit breaker limit silently — always escalate to user
- 🎯 Track loop count per story independently

---

## CORRECTIVE LOOP ALGORITHM

```
FOR each story in stories_needs_changes:
  loop_count = 0

  WHILE status == needs_changes AND loop_count < max_correction_loops:
    loop_count += 1

    1. Dispatch implementor with review feedback
    2. Implementor applies fixes + commits
    3. Dispatch reviewer to re-review
    4. IF approved → EXIT loop (story moves to approved)
    5. IF needs_changes → CONTINUE loop
    6. IF rejected → ESCALATE immediately

  IF loop_count >= max_correction_loops AND status != approved:
    → ESCALATE to human
```

---

## EXECUTION SEQUENCE

### 0. Handle Skip Case

IF this step was skipped (all stories approved in step 6):

Default state for downstream steps:
```yaml
corrective_state:
  correction_tracking: []
  total_correction_loops: 0
  stories_approved: "{from review_state.stories_approved}"
  stories_force_approved: []
  stories_dropped: []
```

Downstream steps should use this default corrective_state when step 7 was not executed.

### 1. Initialize Loop Tracking

```yaml
correction_tracking:
  - story_id: "{story-id}"
    loop_count: 0
    max_loops: "{max_correction_loops}"
    status: "needs_changes"
    history:
      - iteration: 0
        reviewer_findings: {original review findings}
```

### 2. Dispatch Corrective Implementor

For each story needing changes, dispatch implementor via Task tool with ENHANCED contract:

```markdown
## Corrective Implementor Contract

**Role:** Story Implementor — CORRECTION MODE
**Story:** {story_file_path}
**Worktree:** {worktree_path}
**Branch:** {branch_name}
**Correction Iteration:** {loop_count} of {max_correction_loops}

### Context: You are FIXING issues found in code review

Your previous implementation was reviewed and the following issues were found.
You MUST address ALL critical and major findings.

### Review Feedback to Address:

{For each finding in review_report.findings:}
**[{severity}] {category}**: {description}
  File: {file}:{line}
  Suggestion: {suggestion}

### Suggested Improvements:
{review_report.suggested_improvements}

### Working Directory
{worktree_path} — same as before. Your previous commits are here.

### CRITICAL RESTRICTIONS (same as original)
- ✅ You MAY: edit files, create files, delete files, git add, git commit
- 🛑 You MUST NOT: git branch, git checkout, git fetch, git pull, git push

### Exit Criteria for Correction
- ALL critical findings addressed
- ALL major findings addressed
- Minor findings addressed where practical
- Changes committed with message referencing the correction:
  "fix: address review feedback (iteration {loop_count})"
- Tests pass after corrections

### Reporting Format
```yaml
correction_report:
  story_id: "{story-id}"
  iteration: {loop_count}
  status: "completed"  # or "failed" or "blocked"
  findings_addressed:
    - finding: "{description}"
      resolution: "How it was fixed"
  files_changed: list[string]
  commit_hash: "{hash}"
  summary: "What was corrected"
  remaining_concerns: list[string]
```
```

### 3. Verify Correction Commits

After implementor completes, verify new commits exist:

```bash
git -C {worktree_path} log --oneline {base_branch}..HEAD
git -C {worktree_path} diff --stat HEAD~1..HEAD  # changes in latest commit
```

### 4. Re-Dispatch Reviewer

Send the SAME reviewer contract from step 6, with additional context:

```markdown
### Additional Context: This is correction iteration {loop_count}

Previous findings that were addressed:
{list of findings from previous review}

Correction commit(s):
{git log output showing correction commits}

Please verify the findings have been properly addressed AND check for any new issues introduced.
```

### 5. Evaluate Re-Review Result

```
IF review_status == "approved":
  → Move story to stories_approved
  → Log: "Story {id} approved after {loop_count} correction(s)"
  → EXIT loop for this story

IF review_status == "needs_changes":
  → Increment loop_count
  → IF loop_count < max_correction_loops:
    → Record new findings in history
    → CONTINUE loop (go to step 2)
  → IF loop_count >= max_correction_loops:
    → ESCALATE (go to step 6)

IF review_status == "rejected":
  → ESCALATE immediately regardless of loop count
```

### 6. Circuit Breaker Escalation

When a story hits the circuit breaker limit:

```
🚨 CIRCUIT BREAKER — Story {story-id}
═══════════════════════════════════════

This story has gone through {max_correction_loops} correction cycles
without achieving reviewer approval.

Correction History:
  Iteration 1: {findings_count} findings → {addressed_count} addressed
  Iteration 2: {findings_count} findings → {addressed_count} addressed
  Iteration 3: {findings_count} findings → {addressed_count} addressed

Remaining Issues:
{list current unresolved findings}

Options:
[F] Force approve — accept current state with known issues
[M] Manual fix — I'll fix it myself, then re-review
[D] Drop story — exclude from this run
[X] Abort workflow
```

**Handle response:**
- **F**: Add to approved with flag `force_approved: true`
- **M**: Pause this story, let user fix, then re-dispatch reviewer only
- **D**: Move to `stories_dropped` list
- **X**: Abort entire workflow (offer cleanup)

### 7. Present Corrective Loop Summary

After all stories have exited the loop (approved, escalated, or dropped):

```
🔄 CORRECTIVE LOOP SUMMARY
═══════════════════════════════════════

Total correction cycles executed: {total_across_all_stories}

  ✅ {story-id-1}: Approved after 1 correction
  ✅ {story-id-2}: Approved after 2 corrections
  ⚠️ {story-id-3}: Force approved (3 corrections, user override)
  ❌ {story-id-4}: Dropped by user

Stories proceeding: {count}
```

### 8. Store Corrective State

```yaml
corrective_state:
  correction_tracking: {complete history}
  total_correction_loops: int
  stories_approved: list[string]  # Updated with newly approved
  stories_force_approved: list[{story_id, remaining_issues}]
  stories_dropped: list[string]
```

### 9. Proceed to Cross-Validation

```
✅ Corrective loop complete.
   {approved_count} stories ready for cross-validation.
   Loading Step 8: Cross-Validation...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ Each story gets up to `{max_correction_loops}` attempts
- ✅ Full review feedback included in each correction dispatch
- ✅ Circuit breaker triggers at the limit — never silently exceeded
- ✅ User always decides on escalated stories
- ✅ Complete correction history tracked

## FAILURE MODES

- ❌ Exceeding circuit breaker limit without escalation
- ❌ Not including review feedback in correction contract
- ❌ Not verifying correction commits exist
- ❌ Auto-approving after circuit breaker (must be user decision)
- ❌ Losing correction history between iterations
