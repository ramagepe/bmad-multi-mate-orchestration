---
name: step-14-summary-report
description: "Generate execution report for the entire orchestration run"
phase: sequential
phase_number: 3
executor: orchestrator
final_step: true
---

# Step 14: Summary Report

**Progress: Step 14 of 14** — FINAL STEP
**Phase:** 3 (SEQUENTIAL — Orchestrator Only)

---

## STEP GOAL

Generate a comprehensive execution report covering the entire orchestration run. This fulfills the output contract and provides the user with a complete record of what happened, what succeeded, what failed, and what actions remain.

---

## MANDATORY EXECUTION RULES

- 🛑 Include ALL information from all steps — this is the final record
- 📖 Be precise with numbers — verify against stored state
- 🎯 Fulfill the complete output contract

---

## EXECUTION SEQUENCE

### 1. Compile Output Contract

Assemble the output contract from all step states:

```yaml
output:
  execution_summary:
    stories_completed: list    # Stories that were approved + pushed
    stories_failed: list       # Stories that failed implementation
    stories_pending_review: list  # Stories approved but not pushed
  branches_created: list[string]   # All branches created during workflow
  branches_pushed: list[string]    # Branches successfully pushed
  actions_pending_approval: list   # Any remaining unpushed branches
  shared_file_mutations: list      # All detected mutations and resolutions
  correction_loops_executed: int   # Total correction cycles across all stories
```

### 2. Generate Execution Timeline

```
📊 EXECUTION REPORT — orchestrate-implementation
═══════════════════════════════════════════════════

Run: {timestamp_start} → {timestamp_end}
Duration: {total_duration}
Epic: {epic_path}
Base Branch: {base_branch} @ {base_commit_short}

═══════════════════════════════════════════════════

📋 PHASE 1: PREPARATION
───────────────────────────────────────

Step 1 — Readiness Check:
  Stories analyzed: {total}
  Ready: {ready_count} | Partial: {partial_count} | Not Ready: {not_ready_count}
  Proceeded with: {proceeding_count} stories

Step 2 — Shared File Snapshot:
  Files tracked: {snapshot_count}
  Cross-story overlaps: {overlap_count}
  Dependency warnings: {yes/no}

Step 3 — Worktree Setup:
  Worktrees created: {success_count}/{total_count}
  {if failures} Failed: {list} {/if}

═══════════════════════════════════════════════════

🔧 PHASE 2: IMPLEMENTATION
───────────────────────────────────────

Step 4 — Implementor Fan-Out:
  Dispatched: {total} sub-agents
  Max parallel: {max_parallel_agents}
  Completed: {completed} | Failed: {failed} | Timeout: {timeout}

Step 5 — Output Collection:
  Ready for review: {count}
  Needs correction: {count}
  Failed: {count}
  Uncommitted work: {count}

═══════════════════════════════════════════════════

🔍 PHASE 3: REVIEW & VALIDATION
───────────────────────────────────────

Step 6 — Code Review:
  Approved: {count} | Needs Changes: {count} | Rejected: {count}

Step 7 — Corrective Loop:
  Total correction cycles: {total_loops}
  Stories corrected successfully: {count}
  Stories force-approved: {count}
  Stories dropped: {count}
  Circuit breaker triggered: {count} times

Step 8 — Cross-Validation:
  Status: {coherent / issues_found}
  Conflicts detected: {count}
  Gaps found: {count}
  {if issues} Key issues: {list} {/if}

Step 9 — Shared File Mutations:
  Mutations detected: {count}
  Mutations reverted: {count}
  Mutations accepted: {count}
  Multi-story conflicts: {count}

═══════════════════════════════════════════════════

### 2.5 Test Quality Summary

```
🧪 TEST METRICS
═══════════════════════════════════════

Per-Story Test Results:
  {for each story:}
  {story-id}: {pass/fail} — {test_count} tests, Coverage: {if available}
  {/for}

Aggregate:
  Total test verifications: {count}
  Independent verification passed: {count}
  Test discrepancies detected: {count} (sub-agent reported pass, tests actually failed)
  Pre-merge test regression: {count}
  
{if test_discrepancies > 0}
⚠️ Test discrepancies were detected — sub-agent reports were inaccurate
   for {count} stories. Review findings in Step 5 collection report.
{/if}
```

═══════════════════════════════════════════════════

🚦 PHASE 4: MERGE & DELIVERY
───────────────────────────────────────

Step 10 — Pre-Merge Gate:
  Clean merges: {count}
  Conflict risk: {count}
  Base branch moved: {yes/no}

Step 11 — Human Approval:
  Approved for push: {count}
  Skipped: {count}

Step 12 — Push:
  Successfully pushed: {count}
  Failed to push: {count}
  Branches on remote: {list}

Step 13 — Cleanup:
  Worktrees removed: {count}
  Worktrees preserved: {count}
  Branches cleaned: {count}

═══════════════════════════════════════════════════
```

### 3. Story-Level Detail

For each story, provide a complete trace:

```
📖 STORY DETAIL
═══════════════════════════════════════════════════

{for each story:}
───────────────────────────────────────
{story-id}: {story_title}
───────────────────────────────────────
  Readiness: {ready/partial/not_ready}
  Implementation: {completed/failed/timeout}
  Review: {approved/needs_changes/rejected/force_approved}
  Correction loops: {count}
  Cross-validation issues: {count}
  Shared mutations: {count}
  Merge status: {clean/conflict_risk}
  Approval: {approved/skipped}
  Push: {pushed/not_pushed/failed}
  Branch: {branch_name}
  Commits: {count}
  Files changed: {count}
  {if preserved} Worktree preserved at: {path} {/if}
{/for}
```

### 4. Pending Actions

If anything remains to be done:

```
📋 PENDING ACTIONS
═══════════════════════════════════════════════════

{if unpushed_branches}
🔀 Unpushed branches (local only):
  {for each}
  • {branch_name} — {reason not pushed}
    To push: git push origin {branch_name}
    Worktree: {path}
  {/for}
{/if}

{if force_approved}
⚠️ Force-approved stories with known issues:
  {for each}
  • {story-id}: {remaining_issues}
  {/for}
{/if}

{if cross_validation_issues}
🔀 Cross-validation issues to address:
  {for each}
  • {issue_description}
  {/for}
{/if}

{if pr_creation_needed}
💡 Create Pull Requests:
  {for each pushed branch}
  gh pr create --head {branch_name} --base {base_branch} --title "feat: {story_title}"
  {/for}
{/if}
```

### 5. Final Summary

```
═══════════════════════════════════════════════════
🏁 ORCHESTRATION COMPLETE
═══════════════════════════════════════════════════

Result: {total_pushed}/{total_stories} stories pushed to remote
Correction loops: {total_loops} across all stories
Shared mutations: {total_mutations} detected, {total_resolved} resolved

{if all_success}
🎉 All stories implemented, reviewed, approved, and pushed!
{else}
📋 {pending_count} items require follow-up action (see above)
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
    stories_completed: ["{pushed story ids}"]
    stories_failed: ["{failed story ids}"]
    stories_pending_review: ["{unpushed story ids}"]
  branches_created: ["{all branch names}"]
  branches_pushed: ["{pushed branch names}"]
  actions_pending_approval: ["{pending items}"]
  shared_file_mutations: ["{mutation details}"]
  correction_loops_executed: {total_count}
  test_metrics:
    total_verifications: int
    independently_passed: int
    discrepancies_detected: int
    pre_merge_regressions: int
```

---

## SUCCESS METRICS

- ✅ Complete output contract fulfilled
- ✅ Every step's results accurately reported
- ✅ Story-level detail for each story
- ✅ Pending actions clearly listed
- ✅ PR creation hints provided for pushed branches

## FAILURE MODES

- ❌ Missing steps in the report
- ❌ Incorrect numbers (verify against stored state)
- ❌ Not listing pending actions
- ❌ Not providing PR creation hints for pushed branches
