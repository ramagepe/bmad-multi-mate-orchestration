---
name: step-07-human-escalation
description: "Present unresolved stories to user for decision after circuit breaker exhaustion"
nextStepFile: './step-08-summary-report.md'
phase: sequential
phase_number: 3
executor: orchestrator
---

# Step 7: Human Escalation

**Progress: Step 7 of 8** — Next: Summary Report
**Phase:** 3 (SEQUENTIAL — Human Decision Point)

---

## STEP GOAL

Present stories that exhausted the corrective loop circuit breaker to the user for decision. The user decides the final disposition of each unresolved story: force approve, manual fix, or drop. This ensures no story is silently abandoned and the user has full visibility into remaining issues.

---

## MANDATORY EXECUTION RULES

- 🛑 NEVER auto-decide on unresolved stories — the user MUST decide each one
- 📖 Present the COMPLETE correction history for each story
- 🚫 NEVER skip this step if there are stories for escalation
- 🎯 Each escalated story gets its own individual decision
- ⏱️ No timeout on user decisions — wait indefinitely

---

## EXECUTION SEQUENCE

### 0. Handle Skip Case

IF there are no stories for escalation (`stories_for_escalation` is empty):

This step should not have been reached — but if it is:

```
ℹ️ No stories require escalation. All issues were resolved.
   Loading Step 8: Summary Report...
```

Load `{nextStepFile}`.

### 1. Present Escalation Overview

```
🚨 HUMAN ESCALATION — {escalation_count} stories need your decision
═══════════════════════════════════════

These stories went through {max_correction_loops} correction cycles
without fully resolving cross-validation issues.

You must decide the disposition of each story.

⚠️ DEFAULT: No story will be force-approved without your explicit decision.
```

### 2. Individual Story Escalation

For EACH story in `{stories_for_escalation}`:

```
───────────────────────────────────────
🚨 ESCALATION: {story-id}
───────────────────────────────────────

Story file: {stories_output_path}/{story_key}.md
Correction Attempts: {loop_count} of {max_correction_loops}

📜 CORRECTION HISTORY:

Iteration 1:
  Issues found: {issue_count}
  {For each issue:}
    [{severity}] {type}: {description}
  Resolution attempted: {correction_report.summary}
  Result: {issues remaining after correction}

Iteration 2:
  Issues found: {issue_count}
  {For each issue:}
    [{severity}] {type}: {description}
  Resolution attempted: {correction_report.summary}
  Result: {issues remaining after correction}

Iteration 3:
  Issues found: {issue_count}
  {For each issue:}
    [{severity}] {type}: {description}
  Resolution attempted: {correction_report.summary}
  Result: {issues remaining after correction}

📋 REMAINING UNRESOLVED ISSUES:
  {For each unresolved issue:}
  • [{severity}] {type}: {description}
    Related stories: {story_ids}
    Recommendation: {validator recommendation}

📄 CURRENT STORY FILE:
  {stories_output_path}/{story_key}.md — {file_size}

───────────────────────────────────────

What would you like to do with {story-id}?

[F] Force approve — accept current artifacts with known issues
[M] Manual fix — I'll edit the artifacts myself, then continue
[R] Re-process — one more attempt with additional guidance from me
[D] Drop story — exclude from final output
[V] View artifacts — show me the current artifact content
```

### 3. Handle User Decision per Story

**[F] Force Approve:**
```
⚠️ Force approving {story-id} with known issues:
{list of unresolved issues}

These issues will be noted in the final summary report.
Confirm? [Y/N]
```

IF confirmed:
- Add to `stories_force_approved` with `force_approved: true`
- Record remaining issues for the summary report

**[M] Manual Fix:**
```
📝 Manual fix mode for {story-id}.

Story file: {stories_output_path}/{story_key}.md

Edit the file as needed, then tell me when you're done.
I'll re-run cross-validation on this story to verify.

[DONE] I've finished my edits — re-validate
[CANCEL] Cancel — choose a different option
```

IF DONE:
- Re-dispatch targeted cross-validator for this story
- IF coherent → move to resolved
- IF issues_found → present results and ask user again (F/M/D)

**[R] Re-process with Guidance:**
```
📝 Provide additional guidance for the document processor.
What specific instructions should the sub-agent follow?

> {user types guidance}
```

- Dispatch processor with user's guidance PLUS previous correction feedback
- Re-validate after completion
- IF coherent → move to resolved
- IF issues_found → present results and ask user again (F/M/D)
- NOTE: This does NOT reset the circuit breaker — it's a one-time extra attempt

**[D] Drop Story:**
```
❌ Dropping {story-id} from this orchestration run.

The existing story file at {stories_output_path}/{story_key}.md will be preserved
but NOT included in the final summary as completed.

Confirm? [Y/N]
```

IF confirmed:
- Add to `stories_dropped`
- Artifacts remain on disk but are excluded from results

**[V] View Artifacts:**
- Read and display the content of the story file at `{stories_output_path}/{story_key}.md`
- After viewing, re-present the decision menu for this story

### 4. Present Escalation Summary

After all escalated stories have a decision:

```
🚨 ESCALATION SUMMARY
═══════════════════════════════════════

  ✅ {story-id-1}: Force approved (known issues noted)
  ✅ {story-id-2}: Resolved via manual fix
  ✅ {story-id-3}: Resolved via re-processing with guidance
  ❌ {story-id-4}: Dropped by user

Stories proceeding: {count}
Stories dropped: {count}
```

### 5. Store Escalation State

```yaml
escalation_state:
  stories_force_approved: list[{story_id, remaining_issues}]
  stories_manually_resolved: list[string]
  stories_reprocessed: list[{story_id, user_guidance}]
  stories_dropped: list[{story_id, reason: "user_decision"}]
  all_decisions_made: true
```

### 6. Proceed to Summary Report

```
✅ All escalations resolved.
   Loading Step 8: Summary Report...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ Every escalated story received an individual user decision
- ✅ Complete correction history presented before each decision
- ✅ User had option to view artifacts before deciding
- ✅ Force-approved stories have known issues documented
- ✅ No story auto-decided without user input

## FAILURE MODES

- ❌ **CRITICAL**: Auto-approving or auto-dropping stories without user decision
- ❌ Not presenting the full correction history
- ❌ Not offering the "view artifacts" option
- ❌ Not re-validating after manual fix or re-processing
- ❌ Rushing user through decisions
- ❌ Losing track of which stories were force-approved vs resolved
