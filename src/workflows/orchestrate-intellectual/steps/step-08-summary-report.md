---
name: step-08-summary-report
description: "Generate execution report for the entire intellectual orchestration run"
phase: sequential
phase_number: 3
executor: orchestrator
final_step: true
---

# Step 8: Summary Report

**Progress: Step 8 of 8** — FINAL STEP
**Phase:** 3 (SEQUENTIAL — Orchestrator Only)

---

## STEP GOAL

Generate a comprehensive execution report covering the entire intellectual orchestration run. This fulfills the output contract and provides the user with a complete record of what happened, what succeeded, what failed, and what actions remain.

---

## MANDATORY EXECUTION RULES

- 🛑 Include ALL information from all steps — this is the final record
- 📖 Be precise with numbers — verify against stored state
- 🎯 Fulfill the complete output contract
- 📋 List every artifact produced with its location

---

## EXECUTION SEQUENCE

### 1. Compile Output Contract

Assemble the output contract from all step states:

```yaml
output:
  execution_summary:
    stories_completed: list        # Stories with approved artifacts
    stories_failed: list           # Stories that failed processing
    stories_pending_review: list   # Stories force-approved with known issues
    stories_excluded: list         # Stories excluded at readiness check (step 1)
    stories_dropped: list          # Stories dropped at escalation (step 7)
  validation_status: enum          # coherent | issues_found | issues_force_accepted | skipped | skipped_no_stories
  correction_loops_executed: int   # Total correction cycles across all stories
  actions_pending_user_approval: list  # Any remaining items
  artifacts_produced:
    - story_id: string
      artifact_path: string
      artifact_type: string
```

### 2. Generate Execution Report

```
📊 EXECUTION REPORT — orchestrate-intellectual
═══════════════════════════════════════════════════

Run: {timestamp_start} → {timestamp_end}
Duration: {total_duration}
Epic: {epic_path}
Output Folder: {output_folder}
Mode: Intellectual (document artifacts only)

═══════════════════════════════════════════════════

📋 PHASE 1: PREPARATION
───────────────────────────────────────

Step 1 — Readiness Check:
  Stories analyzed: {total}
  Ready: {ready_count} | Partial: {partial_count} | Not Ready: {not_ready_count}
  Proceeded with: {proceeding_count} stories
  Cross-story concerns flagged: {concern_count}

Step 2 — Fan-Out Planning:
  Total batches planned: {batch_count}
  Workflows assigned:
    {For each unique workflow:}
    • {workflow_name}: {count} stories
  Dependencies resolved: {dependency_count}

═══════════════════════════════════════════════════

🔧 PHASE 2: DOCUMENT PROCESSING
───────────────────────────────────────

Step 3 — Sub-Agent Dispatch:
  Dispatched: {total} sub-agents across {batch_count} batches
  Max parallel: {max_parallel_agents}
  Completed: {completed} | Failed: {failed} | Timeout: {timeout}

Step 4 — Output Collection:
  Successfully processed: {count}
  Needs correction: {count}
  Failed: {count}
  Total artifacts produced: {total_artifact_count}

═══════════════════════════════════════════════════

🔍 PHASE 3: VALIDATION & DELIVERY
───────────────────────────────────────

Step 5 — Cross-Validation:
  Status: {coherent / issues_found}
  Contradictions detected: {count}
  Coverage gaps: {count}
  Terminology issues: {count}
  Dependency issues: {count}
  {if validation_skipped} ⚠️ Validation was skipped by user {/if}

Step 6 — Corrective Loop:
  Total correction cycles: {total_loops}
  Stories resolved via correction: {count}
  Stories sent to escalation: {count}
  Circuit breaker triggered: {count} times

Step 7 — Human Escalation:
  Stories escalated: {count}
  Force approved: {count}
  Manually resolved: {count}
  Re-processed: {count}
  Dropped: {count}
  {if no escalation} ℹ️ No escalation needed — all resolved automatically {/if}

═══════════════════════════════════════════════════
```

### 3. Artifact Inventory

List all produced artifacts:

```
📄 ARTIFACT INVENTORY
═══════════════════════════════════════════════════

{For each story:}
───────────────────────────────────────
{story-id}: {story title}
───────────────────────────────────────
  Status: {completed / force_approved / dropped}
  Output path: {output_path}
  Artifacts:
    {For each artifact:}
    • {artifact_filename} ({artifact_type}) — {file_size}
      {brief description}
  AC Coverage: {percentage}%
  {if force_approved}
  ⚠️ Force approved with known issues:
    {list remaining issues}
  {/if}
  {if correction_loops > 0}
  Correction loops: {count}
  {/if}

{/For}

───────────────────────────────────────
Cross-Validation Report:
  Location: {output_folder}/_orchestration/cross-validation-report.md
  {if created} ✅ Written {/if}
  {if not created} ⚠️ Not generated {/if}
```

### 4. Story-Level Detail

For each story, provide a complete trace:

```
📖 STORY DETAIL
═══════════════════════════════════════════════════

{For each story in original stories list:}
───────────────────────────────────────
{story-id}: {story title}
───────────────────────────────────────
  Readiness: {ready / partial / not_ready / excluded}
  Workflow: {workflow_to_invoke}
  Task type: {task_type}
  Processing: {completed / failed / timeout}
  AC Coverage: {total_acs_addressed}/{total_acs} ({percentage}%)
  Cross-validation: {coherent / issues_found / not_validated}
  Correction loops: {count}
  Final status: {completed / force_approved / dropped / excluded / failed}
  Artifacts: {artifact_count} files in {output_path}
  {if issues}
  Known issues:
    {list remaining issues}
  {/if}
{/For}
```

### 5. Pending Actions

If anything remains to be done:

```
📋 PENDING ACTIONS
═══════════════════════════════════════════════════

{if force_approved_stories}
⚠️ Force-approved stories with known issues:
  {For each:}
  • {story-id}: {remaining_issues_summary}
    Artifacts: {output_path}
    Action needed: Manual review and correction of flagged issues
  {/For}
{/if}

{if dropped_stories}
❌ Dropped stories (not processed or incomplete):
  {For each:}
  • {story-id}: {reason}
    Action needed: Fix story spec and re-run, or process manually
  {/For}
{/if}

{if coverage_gaps}
🟡 Epic coverage gaps identified in cross-validation:
  {For each gap:}
  • {gap_description}
    Action needed: Create additional stories or expand existing ones
  {/For}
{/if}

{if excluded_stories}
📝 Stories excluded at readiness check:
  {For each:}
  • {story-id}: {exclusion_reason}
    Action needed: Address readiness issues and re-run
  {/For}
{/if}

{if no pending actions}
✅ No pending actions — all stories processed successfully!
{/if}
```

### 6. Save Cross-Validation Report to Disk

If a cross-validation report was generated, write it to the orchestration output:

```bash
# Write cross-validation report
# Path: {output_folder}/_orchestration/cross-validation-report.md
```

Write the cross-validation report content to `{output_folder}/_orchestration/cross-validation-report.md` using the Write tool.

### 7. Final Summary

```
═══════════════════════════════════════════════════
🏁 INTELLECTUAL ORCHESTRATION COMPLETE
═══════════════════════════════════════════════════

Result: {completed_count}/{total_stories} stories completed successfully
Artifacts: {total_artifact_count} files in {output_folder}
Correction loops: {total_loops} across all stories
Force approved: {force_approved_count} stories with known issues

{if all_success}
🎉 All stories processed, validated, and coherent!
{else if mostly_success}
📋 {completed_count} stories complete. {pending_count} items require follow-up (see above).
{else}
⚠️ Partial completion. See pending actions above for next steps.
{/if}

Thank you, {user_name}! 🧉
═══════════════════════════════════════════════════
```

---

## OUTPUT CONTRACT FULFILLMENT

This step produces the final output matching the workflow's output contract:

```yaml
output:
  execution_summary:
    stories_completed: ["{completed story ids}"]
    stories_failed: ["{failed story ids}"]
    stories_pending_review: ["{force-approved story ids}"]
    stories_excluded: ["{excluded at readiness check}"]
    stories_dropped: ["{dropped at escalation}"]
  validation_status: "{coherent | issues_found | issues_force_accepted | skipped | skipped_no_stories}"
  correction_loops_executed: {total_count}
  actions_pending_user_approval: ["{pending items}"]
  artifacts_produced:
    - story_id: "{story-id}"
      artifact_path: "{path}"
      artifact_type: "{type}"
```

---

## SUCCESS METRICS

- ✅ Complete output contract fulfilled
- ✅ Every step's results accurately reported
- ✅ Story-level detail for each story
- ✅ Artifact inventory with file locations
- ✅ Pending actions clearly listed
- ✅ Cross-validation report saved to disk

## FAILURE MODES

- ❌ Missing steps in the report
- ❌ Incorrect numbers (verify against stored state)
- ❌ Not listing all produced artifacts
- ❌ Not listing pending actions
- ❌ Not saving cross-validation report to disk
- ❌ Not reporting force-approved stories with their known issues
