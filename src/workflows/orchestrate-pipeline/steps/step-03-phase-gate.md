---
name: step-03-phase-gate
description: "Present intellectual results, get user approval to proceed to implementation"
nextStepFile: './step-04-implementation-phase.md'
skipToStepFile: './step-05-final-summary.md'
skipCondition: "user aborted pipeline at phase gate"
phase: sequential
phase_number: 2
executor: orchestrator
security: critical
max_intellectual_reruns: 2
---

# Step 3: Phase Gate — Intellectual to Implementation

**Progress: Step 3 of 5** — Next: Implementation Phase
**Phase:** 2 (SEQUENTIAL — Human Decision Point)
**Security Level:** CRITICAL — User must approve phase transition

---

## STEP GOAL

Present the complete intellectual phase results to the user and obtain explicit approval before transitioning to the implementation phase. This is the CRITICAL human decision point — the user reviews what was refined, what failed, what was dropped, and decides if the quality of intellectual outputs is sufficient to proceed to code implementation. The user should never be surprised by what enters the implementation phase.

---

## MANDATORY EXECUTION RULES

- 🛑 **ABSOLUTE RULE**: NEVER auto-proceed to implementation — user MUST explicitly approve
- 📖 Present COMPLETE intellectual results — do not summarize away failures or issues
- 🚫 NEVER pressure user to proceed — aborting or re-running is always valid
- 🎯 Show exactly WHICH stories will enter implementation and which will not
- ⚠️ Force-approved stories with known issues MUST be highlighted prominently

---

## EXECUTION SEQUENCE

### 0. State Checkpoint

**⚠️ STATE CHECKPOINT:** Before proceeding, confirm you have these critical pipeline values:
- `epic_path`: {pipeline_state.epic_path}
- `stories` (original list): {pipeline_state.stories}
- `base_branch`: {pipeline_state.base_branch}
- `parallel_limit`: {pipeline_state.parallel_limit}
- `intellectual_output`: {captured from step 2}
- `intellectual_phase_result.classification`: {from step 2}
- `intellectual_phase_result.stories_available_for_implementation`: {from step 2}
If any value is missing, scroll back to pipeline steps 1-2 to recover it before proceeding.

### 1. Present Intellectual Phase Results

Display a comprehensive summary of what the intellectual phase produced:

```
═══════════════════════════════════════════════════
🚦 PHASE GATE — Intellectual → Implementation
═══════════════════════════════════════════════════

The intellectual phase has completed. Review the results below
before deciding whether to proceed to implementation.

───────────────────────────────────────────────────
📊 INTELLECTUAL PHASE SUMMARY
───────────────────────────────────────────────────

Duration: {intellectual phase duration}
Validation Status: {validation_status}
Correction Loops: {correction_loops_executed}

┌──────────────────┬──────────┬─────────────────────────────┐
│ Story            │ Status   │ Notes                       │
├──────────────────┼──────────┼─────────────────────────────┤
│ {story-id-1}     │ ✅ Done  │ All ACs refined             │
│ {story-id-2}     │ ⚠️ Force │ Known issues (see below)    │
│ {story-id-3}     │ ❌ Failed│ {failure reason}            │
│ {story-id-4}     │ 🚫 Drop  │ Dropped at escalation       │
│ {story-id-5}     │ ⏭️ Excl  │ Excluded at readiness       │
└──────────────────┴──────────┴─────────────────────────────┘

Artifacts Produced: {artifact_count} files in {output_folder}
```

### 2. Highlight Issues and Risks

If there are force-approved stories, failed stories, or cross-validation issues, present them prominently:

```
{if force_approved_stories}
⚠️ FORCE-APPROVED STORIES — Proceeding with known issues:
───────────────────────────────────────────────────
{For each force-approved story:}
  • {story-id}: {remaining_issues_summary}
    Artifacts: {artifact_path}
    Risk: These issues may cascade into implementation
{/For}
{/if}

{if validation_issues}
⚠️ CROSS-VALIDATION ISSUES — Found during intellectual phase:
───────────────────────────────────────────────────
{For each issue:}
  • {issue_type}: {issue_description}
    Affected stories: {story_ids}
{/For}
{/if}

{if failed_or_dropped_stories}
❌ STORIES NOT AVAILABLE FOR IMPLEMENTATION:
───────────────────────────────────────────────────
{For each failed/dropped/excluded story:}
  • {story-id}: {status} — {reason}
{/For}

These stories will NOT enter the implementation phase.
{/if}
```

### 3. Present Implementation Preview

Show what will happen if the user proceeds:

```
───────────────────────────────────────────────────
🔜 IMPLEMENTATION PHASE PREVIEW
───────────────────────────────────────────────────

Stories entering implementation: {available_count}
{For each available story:}
  • {story-id} — using refined artifacts from intellectual phase
{/For}

Base Branch: {base_branch}
Worktree Path: {worktree_base_path}
Parallel Limit: {parallel_limit}

The implementation phase will:
  • Create git worktrees for each story
  • Dispatch implementor sub-agents in parallel
  • Run code review per story
  • Cross-validate across all implementations
  • Present merge candidates for your approval

Estimated sub-agents: {available_count} implementors + {available_count} reviewers
```

### 4. User Decision Gate

```
═══════════════════════════════════════════════════
📋 DECISION REQUIRED
═══════════════════════════════════════════════════

{if all_stories_available}
All {available_count} stories are ready for implementation.
{else}
{available_count} of {total_count} stories are available for implementation.
{unavailable_count} stories will be skipped (failed/dropped/excluded).
{/if}

───────────────────────────────────────────────────

[P] Proceed to implementation with {available_count} stories
[S] Select specific stories — choose which ones to implement
[R] Re-run intellectual phase — try again with all stories
[A] Abort pipeline — stop here, keep intellectual artifacts

═══════════════════════════════════════════════════
```

### 5. Handle User Decision

**[P] Proceed:**
- Confirm the stories entering implementation
- Update pipeline state with the implementation story list
- Proceed to step 4

**[S] Select specific stories:**
```
Select stories for implementation (enter story IDs, comma-separated):

Available stories:
{For each available story:}
  [{n}] {story-id} — {status from intellectual phase}
{/For}

Enter selections (e.g., 1,3,4) or [B] to go back:
```
- Validate selections
- **IF zero valid stories selected:** Report "No valid stories selected. You must select at least one story to proceed." Return to decision gate.
- Update pipeline state with selected stories only
- Return to decision gate to confirm

**[R] Re-run intellectual phase:**

**Circuit breaker check:** Track `intellectual_rerun_count` in pipeline state. Maximum reruns: 2 (from frontmatter `max_intellectual_reruns`).

**IF `intellectual_rerun_count >= max_intellectual_reruns`:**
```
🛑 Maximum intellectual re-runs reached ({max_intellectual_reruns}).
   The intellectual phase has been run {intellectual_rerun_count + 1} times total.
   
   Options:
   [P] Proceed with current results anyway
   [A] Abort pipeline
```

**IF reruns available:**
```
⚠️ Re-running the intellectual phase will:
  • Overwrite existing intellectual artifacts
  • Start from intellectual step 1 (readiness check)
  • This is re-run {intellectual_rerun_count + 1} of {max_intellectual_reruns} allowed
  • ⚠️ Re-running consumes significant context — consider aborting and restarting fresh
  
Confirm re-run? [Y/N]
```
- **Y**: Increment `intellectual_rerun_count`. Return to step 2 (intellectual phase) — re-execute from the beginning
- **N**: Return to decision gate

**[A] Abort pipeline:**
```
ℹ️ Pipeline aborted at phase gate.
   
   Intellectual artifacts are preserved at: {output_folder}
   No implementation phase will run.
   
   To resume later:
   • Use [OD] from orchestrator menu to run implementation independently
   • Stories will need implementation-specific readiness check
```
STOP pipeline.

### 6. Update Pipeline State for Implementation

After user approves, update the pipeline state:

```yaml
pipeline_state:
  phase: "starting_implementation"
  stories_for_implementation: list   # Stories user approved for implementation
  stories_skipped_at_gate: list      # Stories user chose to skip
  intellectual_output: "{captured}"   # Preserved for final summary
  gate_decision: "proceed"           # proceed | rerun | abort
  gate_timestamp: "{timestamp}"
  intellectual_rerun_count: int      # Number of times intellectual phase was re-run (0 if never)
```

### 7. Proceed to Implementation Phase

```
═══════════════════════════════════════════════════
✅ PHASE GATE APPROVED
═══════════════════════════════════════════════════

Proceeding to implementation with {implementation_count} stories.

{if stories_skipped_at_gate}
Skipped at gate:
{For each skipped:}
  ⏭️ {story-id} — {reason}
{/For}
{/if}

Loading Step 4: Implementation Phase...
═══════════════════════════════════════════════════
```

Load, read completely, then execute `{nextStepFile}`.

---

## QUALITY GATE: PHASE TRANSITION

| Validates | Failure Action |
|-----------|----------------|
| At least one story available for implementation | Cannot proceed — re-run or abort |
| User explicitly approved phase transition | Wait for user decision — never auto-proceed |
| User informed of all risks and issues | Present everything before asking for decision |

---

## SUCCESS METRICS

- ✅ Complete intellectual results presented to user
- ✅ Force-approved stories and their issues highlighted
- ✅ Cross-validation issues displayed
- ✅ Implementation preview shown with story list and branch info
- ✅ User made informed decision (proceed/select/rerun/abort)
- ✅ Pipeline state updated with implementation story list

## FAILURE MODES

- ❌ **CRITICAL**: Auto-proceeding to implementation without user approval
- ❌ Hiding or minimizing force-approved story issues
- ❌ Not showing which stories will NOT enter implementation
- ❌ Not offering the option to select specific stories
- ❌ Not offering re-run as an option
- ❌ Pressuring user to proceed
