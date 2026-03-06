---
name: step-05-final-summary
description: "Combined report from both intellectual and implementation phases"
phase: sequential
phase_number: 3
executor: orchestrator
final_step: true
---

# Step 5: Final Summary

**Progress: Step 5 of 5** — FINAL STEP
**Phase:** 3 (SEQUENTIAL — Orchestrator Only)

---

## STEP GOAL

Generate a comprehensive combined execution report covering BOTH the intellectual and implementation phases. This fulfills the pipeline's output contract and provides the user with a single unified record of the entire end-to-end pipeline run — what was refined, what was implemented, what succeeded, what failed, and what actions remain across both phases.

---

## MANDATORY EXECUTION RULES

- 🛑 Include results from BOTH phases — this is the combined record
- 📖 Be precise with numbers — verify against stored state from both phases
- 🎯 Fulfill the pipeline's output contract (different from either sub-workflow's contract)
- 📋 List ALL artifacts from intellectual phase AND all branches from implementation phase
- ⚠️ Clearly show the story journey: intellectual result → gate decision → implementation result

---

## EXECUTION SEQUENCE

### 0. State Checkpoint

**⚠️ STATE CHECKPOINT:** Before generating the final report, confirm you have these values:
- `pipeline_state` from step 1 (epic_path, original stories, base_branch)
- `intellectual_output` from step 2 (execution_summary, validation_status, artifacts_produced)
- `gate_decision` and `stories_for_implementation` from step 3
- `implementation_output` from step 4 (execution_summary, branches, test_metrics)
- `pipeline_start_timestamp` from step 1
If any value is missing, scroll back to the relevant pipeline step to recover it.

### 1. Compile Pipeline Output Contract

Assemble the pipeline output contract from both phase outputs:

```yaml
output:
  execution_summary:
    intellectual_phase:
      stories_refined: list          # Stories completed in intellectual phase
      stories_excluded: list         # Excluded at intellectual readiness
      stories_dropped: list          # Dropped at intellectual escalation
      cross_validation_status: string # coherent | issues_found | issues_force_accepted | skipped
      artifacts_produced: list       # Document artifacts from intellectual phase
    implementation_phase:
      stories_completed: list        # Stories pushed to remote
      stories_failed: list           # Stories that failed implementation
      stories_excluded: list         # Excluded at implementation readiness
      stories_dropped: list          # Dropped at implementation escalation
      branches_created: list[string] # All branches created
      branches_pushed: list[string]  # Branches successfully pushed to remote
    stories_skipped_at_gate: list    # Stories user chose to skip at phase gate
  actions_pending_approval: list     # Combined pending from both phases
  shared_file_mutations: list        # From implementation phase
  total_correction_loops: int        # Sum from both phases
  test_metrics:                      # From implementation phase
    total_verifications: int
    independently_passed: int
    discrepancies_detected: int
    pre_merge_regressions: int
```

### 2. Generate Combined Execution Report

```
═══════════════════════════════════════════════════
📊 PIPELINE EXECUTION REPORT
═══════════════════════════════════════════════════
orchestrate-pipeline — Full Pipeline Summary

Run: {pipeline_start_timestamp} → {pipeline_end_timestamp}
Total Duration: {total_pipeline_duration}
Epic: {epic_path}
Mode: Mixed (intellectual → implementation)
Original Stories: {original_story_count}

═══════════════════════════════════════════════════

🧠 PHASE 1: INTELLECTUAL — Document Refinement
───────────────────────────────────────────────────

Duration: {intellectual_phase_duration}
Classification: {intellectual_phase_result.classification}

Stories:
  Completed: {completed_count}
  Force-approved: {force_approved_count}
  Failed: {failed_count}
  Dropped: {dropped_count}
  Excluded: {excluded_count}

Validation Status: {validation_status}
Correction Loops: {intellectual_correction_loops}
Artifacts Produced: {artifact_count} files

{if intellectual_issues}
Issues Carried Forward:
  {For each issue:}
  • {issue_description}
  {/For}
{/if}

═══════════════════════════════════════════════════

🚦 PHASE GATE — Intellectual → Implementation
───────────────────────────────────────────────────

Decision: {gate_decision}
Stories approved for implementation: {gate_approved_count}
Stories skipped at gate: {gate_skipped_count}
{if gate_skipped}
Skipped:
  {For each:}
  • {story-id}: {reason}
  {/For}
{/if}

═══════════════════════════════════════════════════

🔧 PHASE 2: IMPLEMENTATION — Code in Worktrees
───────────────────────────────────────────────────

Duration: {implementation_phase_duration}
Classification: {implementation_phase_result.classification}
Base Branch: {base_branch}

Stories:
  Entered implementation: {implementation_story_count}
  Completed + pushed: {pushed_count}
  Completed (not pushed): {unpushed_count}
  Failed: {impl_failed_count}

Branches:
  Created: {branches_created_count}
  Pushed to remote: {branches_pushed_count}
  {For each pushed branch:}
    • {branch_name}
  {/For}

Code Review:
  Approved: {review_approved_count}
  Force-approved: {force_approved_count}
  Correction loops: {impl_correction_loops}

Shared File Mutations: {mutation_count} detected
  {For each mutation:}
  • {file_path}: {resolution}
  {/For}

{if test_metrics}
Test Metrics:
  Verifications: {total_verifications}
  Independently passed: {independently_passed}
  Discrepancies: {discrepancies_detected}
  Pre-merge regressions: {pre_merge_regressions}
{/if}

═══════════════════════════════════════════════════
```

### 3. Story Journey Trace

> **Note:** For pipeline runs with corrections in both phases, per-story detail may be approximate. Prioritize accuracy for final status and branch information; intermediate phase details are best-effort from context reconstruction.

Show each story's complete journey through the pipeline:

```
📖 STORY JOURNEY — End-to-End Trace
═══════════════════════════════════════════════════

{For each story in original stories list:}
───────────────────────────────────────────────────
{story-id}: {story title}
───────────────────────────────────────────────────
  Phase 1 (Intellectual):
    Readiness: {ready / partial / not_ready / excluded}
    Processing: {completed / failed / force_approved / dropped}
    Artifacts: {artifact_count} files at {output_path}
    {if intellectual_issues} Issues: {list} {/if}
  
  Phase Gate:
    Decision: {approved / skipped / not_applicable}
    {if skipped} Reason: {reason} {/if}
  
  Phase 2 (Implementation):
    {if entered_implementation}
    Implementation: {completed / failed / timeout}
    Review: {approved / force_approved / rejected}
    Correction loops: {count}
    Branch: {branch_name}
    Push: {pushed / not_pushed / failed}
    {if pushed} Remote: origin/{branch_name} {/if}
    {else}
    ⏭️ Did not enter implementation — {reason}
    {/if}
  
  Final Status: {end_to_end_status}
    {if all_success} ✅ Refined + Implemented + Pushed
    {else if intellectual_only} 📄 Refined only — not implemented
    {else if failed} ❌ Failed at {phase}
    {/if}

{/For}
═══════════════════════════════════════════════════
```

### 4. Combined Pending Actions

Aggregate pending actions from both phases:

```
📋 PENDING ACTIONS — All Phases
═══════════════════════════════════════════════════

{if intellectual_pending}
🧠 FROM INTELLECTUAL PHASE:
───────────────────────────────────────────────────
{For each:}
  • {story-id}: {pending_action}
    Artifacts: {output_path}
{/For}
{/if}

{if unpushed_branches}
🔀 UNPUSHED BRANCHES (local only):
───────────────────────────────────────────────────
{For each:}
  • {branch_name} — {reason}
    To push: git push origin {branch_name}
{/For}
{/if}

{if force_approved_anywhere}
⚠️ FORCE-APPROVED WITH KNOWN ISSUES:
───────────────────────────────────────────────────
{For each:}
  • {story-id} ({phase}): {remaining_issues}
{/For}
{/if}

{if cross_validation_issues}
🔀 CROSS-VALIDATION ISSUES:
───────────────────────────────────────────────────
{For each:}
  • {issue_description}
{/For}
{/if}

{if shared_mutations_unresolved}
⚠️ SHARED FILE MUTATIONS — Unresolved:
───────────────────────────────────────────────────
{For each:}
  • {file_path}: {mutation_description}
{/For}
{/if}

{if stories_not_implemented}
📝 STORIES NOT IMPLEMENTED:
───────────────────────────────────────────────────
{For each:}
  • {story-id}: {reason} (failed at: {phase})
    Action: {recommended_action}
{/For}
{/if}

{if pr_creation_needed}
💡 CREATE PULL REQUESTS:
───────────────────────────────────────────────────
{For each pushed branch:}
  gh pr create --head {branch_name} --base {base_branch} --title "feat: {story_title}"
{/For}
{/if}

{if no_pending_actions}
✅ No pending actions — all stories refined, implemented, and pushed!
{/if}

═══════════════════════════════════════════════════
```

### 5. Pipeline Statistics

```
📈 PIPELINE STATISTICS
═══════════════════════════════════════════════════

End-to-End:
  Total pipeline duration: {total_duration}
  Original stories: {original_count}
  Fully completed (refined + pushed): {full_success_count}
  Partially completed: {partial_count}
  Not completed: {not_completed_count}

Intellectual Phase:
  Duration: {intellectual_duration}
  Stories processed: {intellectual_processed}
  Artifacts produced: {artifact_count}
  Correction loops: {intellectual_loops}

Phase Gate:
  Stories approved: {gate_approved}
  Stories filtered: {gate_filtered}

Implementation Phase:
  Duration: {implementation_duration}
  Worktrees created: {worktree_count}
  Branches pushed: {pushed_count}
  Correction loops: {implementation_loops}

Combined Totals:
  Total correction loops: {intellectual_loops + implementation_loops}
  Total sub-agent dispatches: {total_dispatches}

═══════════════════════════════════════════════════
```

### 6. Final Summary

```
═══════════════════════════════════════════════════
🏁 FULL PIPELINE COMPLETE
═══════════════════════════════════════════════════

{if all_stories_pushed}
🎉 All {original_count} stories refined, implemented, reviewed, and pushed!
   Branches on remote: {list}
   Next step: Create PRs for review and merge.
{else if mostly_success}
📋 {pushed_count}/{original_count} stories completed end-to-end.
   {pending_count} items require follow-up (see pending actions above).
{else if partial_completion}
⚠️ Partial pipeline completion.
   Intellectual: {intellectual_completed}/{original_count} stories refined.
   Implementation: {pushed_count}/{implementation_count} stories pushed.
   See pending actions above for next steps.
{else}
❌ Pipeline completed with significant issues.
   Review pending actions above and determine next steps.
{/if}

Thank you, {user_name}! 🧉
═══════════════════════════════════════════════════
```

---

## OUTPUT CONTRACT FULFILLMENT

This step produces the final output matching the pipeline's output contract:

```yaml
output:
  execution_summary:
    intellectual_phase:
      stories_refined: ["{completed intellectual story ids}"]
      stories_excluded: ["{excluded at intellectual readiness}"]
      stories_dropped: ["{dropped at intellectual escalation}"]
      cross_validation_status: "{validation_status}"
      artifacts_produced: ["{artifact list from intellectual phase}"]
    implementation_phase:
      stories_completed: ["{pushed story ids}"]
      stories_failed: ["{failed implementation story ids}"]
      stories_excluded: ["{excluded at implementation readiness}"]
      stories_dropped: ["{dropped at implementation escalation}"]
      branches_created: ["{all branch names}"]
      branches_pushed: ["{pushed branch names}"]
    stories_skipped_at_gate: ["{stories user skipped at phase gate}"]
  actions_pending_approval: ["{combined pending items}"]
  shared_file_mutations: ["{mutation details from implementation}"]
  total_correction_loops: {intellectual_loops + implementation_loops}
  test_metrics:
    total_verifications: {from implementation}
    independently_passed: {from implementation}
    discrepancies_detected: {from implementation}
    pre_merge_regressions: {from implementation}
```

---

## SUCCESS METRICS

- ✅ Complete pipeline output contract fulfilled
- ✅ Both phases' results accurately reported
- ✅ Story journey trace shows end-to-end path for every story
- ✅ Combined pending actions from both phases
- ✅ Pipeline statistics (durations, counts, totals)
- ✅ PR creation hints for pushed branches

## FAILURE MODES

- ❌ Missing one phase's results from the combined report
- ❌ Not showing the story journey trace (unique to pipeline)
- ❌ Incorrect combined numbers (double-counting or missing)
- ❌ Not listing combined pending actions
- ❌ Not providing PR creation hints for pushed branches
- ❌ Not distinguishing intellectual-only stories from fully-implemented ones
- ⚠️ Story journey trace is approximate for long pipeline runs — prioritize final status accuracy over intermediate phase detail
