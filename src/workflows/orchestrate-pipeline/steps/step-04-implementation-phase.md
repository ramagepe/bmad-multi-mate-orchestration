---
name: step-04-implementation-phase
description: "Invoke orchestrate-implementation workflow for code implementation in worktrees"
nextStepFile: './step-05-final-summary.md'
phase: sequential
phase_number: 3
executor: orchestrator
---

# Step 4: Implementation Phase

**Progress: Step 4 of 5** — Next: Final Summary
**Phase:** 3 (SEQUENTIAL — Sub-Workflow Invocation)

---

## STEP GOAL

Invoke the orchestrate-implementation workflow to run the complete code implementation pipeline. This step does NOT re-implement the implementation workflow logic — it delegates to the existing 14-step workflow and captures the output contract when complete. The orchestrator manages the lifecycle: start, monitor for completion or failure, and capture results. The story list may be a subset of the original — only stories that passed the intellectual phase AND were approved at the phase gate.

---

## MANDATORY EXECUTION RULES

- 🛑 NEVER re-implement implementation workflow steps — invoke the existing workflow
- 📖 Load and execute the implementation workflow's first step file directly
- 🚫 NEVER skip the implementation workflow's own readiness check (its step 1)
- 🎯 Pass ONLY the stories approved at the phase gate — not the original full list
- ⚠️ Capture the COMPLETE output contract including test metrics and branch information
- 🔐 The implementation workflow has its own human approval gate (step 11) — respect it

---

## EXECUTION SEQUENCE

### 1. Announce Phase Start

```
═══════════════════════════════════════════════════
🔧 PHASE 2: IMPLEMENTATION — Code in Worktrees
═══════════════════════════════════════════════════

Invoking: orchestrate-implementation workflow
Path: {implementation_workflow_path}
Stories: {implementation_count} stories (from phase gate)
Mode: implementation
Base Branch: {base_branch}

This phase will:
  • Validate story implementation readiness (impl step 1)
  • Snapshot shared files for mutation detection (impl step 2)
  • Create git worktrees per story (impl step 3)
  • Dispatch implementor sub-agents in parallel (impl step 4)
  • Collect outputs and verify commits (impl step 5)
  • Dispatch code reviewers (impl step 6)
  • Run corrective loops for review feedback (impl step 7)
  • Cross-validate across implementations (impl step 8)
  • Check shared file mutations (impl step 9)
  • Run pre-merge gate (impl step 10)
  • Human approval per branch (impl step 11)
  • Push approved branches (impl step 12)
  • Clean up worktrees (impl step 13)
  • Generate implementation summary (impl step 14)

Starting implementation workflow...
───────────────────────────────────────────────────
```

### 2. Prepare Implementation Workflow Inputs

Map pipeline state to implementation workflow input contract:

```yaml
implementation_input:
  epic_path: "{pipeline_state.epic_path}"
  stories: "{pipeline_state.stories_for_implementation}"  # Subset from phase gate
  mode: implementation
  parallel_limit: "{pipeline_state.parallel_limit}"
  base_branch: "{pipeline_state.base_branch}"
```

**Important:** The stories list comes from `pipeline_state.stories_for_implementation`, NOT the original `pipeline_state.stories`. This ensures only phase-gate-approved stories enter implementation.

### 3. Execute Implementation Workflow

Load and execute the implementation workflow's first step:

```
Load file: {implementation_workflow_path}/steps/step-01-readiness-check.md
```

**⚠️ STATE CHECKPOINT:** Before invoking the implementation sub-workflow, confirm you have these critical pipeline values:
- `epic_path`: {pipeline_state.epic_path}
- `stories_for_implementation`: {pipeline_state.stories_for_implementation}
- `base_branch`: {pipeline_state.base_branch}
- `parallel_limit`: {pipeline_state.parallel_limit}
- `intellectual_output`: {captured from step 2}
If any value is missing, scroll back to pipeline steps 1-3 to recover it before proceeding.

**CRITICAL:** Execute the implementation workflow's 14 steps following each step's `nextStepFile` chain:

```
step-01-readiness-check.md
  → step-02-shared-file-snapshot.md
    → step-03-worktree-setup.md
      → step-04-implementor-fanout.md
        → step-05-output-collection.md
          → step-06-reviewer-dispatch.md
            → step-07-corrective-loop.md
              → step-08-cross-validation.md
                → step-09-shared-file-mutation-check.md
                  → step-10-pre-merge-gate.md
                    → step-11-human-approval-gate.md
                      → step-12-push-approved-branches.md
                        → step-13-cleanup.md
                          → step-14-summary-report.md (final_step: true — RETURN TO PIPELINE)
```

Each step file is self-contained with its own execution sequence. Follow them exactly as written.

**⚠️ RETURN TO PIPELINE:** When `step-14-summary-report.md` completes (it is marked `final_step: true` and has NO `nextStepFile`), you MUST return here to this pipeline step and continue at sub-step 4 below ("Capture Implementation Output Contract"). Do NOT stop execution — the sub-workflow is complete but the pipeline is not.

**Important behavioral notes:**
- The implementation workflow's step 1 will perform its OWN readiness check — allow it. It may reclassify stories using implementation-specific criteria.
- The implementation workflow's steps 7 and 11 involve user interaction (corrective loop escalation and branch approval) — this is expected.
- The implementation workflow's step 11 is a security-critical approval gate — the user approves each branch for push individually.
- The implementation workflow's step 14 generates a summary report — this is the output we need to capture.

### 4. Capture Implementation Output Contract

After the implementation workflow completes (step 14 summary report), capture its output contract:

```yaml
implementation_output:
  execution_summary:
    stories_completed: list       # Stories approved and pushed
    stories_failed: list          # Stories that failed implementation
    stories_pending_review: list  # Stories approved but not pushed
    stories_excluded: list        # Stories excluded at implementation readiness check
    stories_dropped: list         # Stories dropped at implementation escalation
  branches_created: list[string]
  branches_pushed: list[string]
  actions_pending_approval: list
  shared_file_mutations: list
  correction_loops_executed: int
  test_metrics:
    total_verifications: int
    independently_passed: int
    discrepancies_detected: int
    pre_merge_regressions: int
```

Store this output — it will be included in the final combined summary report (step 5).

### 5. Classify Implementation Phase Result

Based on the output contract, classify the phase result:

| Condition | Classification | Pipeline Action |
|-----------|---------------|-----------------|
| All stories pushed, tests pass | ✅ FULL_SUCCESS | Proceed to summary |
| Some stories pushed, some pending | ⚠️ PARTIAL_SUCCESS | Proceed to summary |
| No stories pushed (all failed or skipped) | ❌ NO_PUSHES | Proceed to summary (report outcome) |
| All stories failed implementation | ❌ TOTAL_FAILURE | Proceed to summary (report failure) |
| User aborted during implementation workflow | 🛑 USER_ABORT | Proceed to summary (report partial) |

**Note:** Unlike the intellectual phase, we ALWAYS proceed to the final summary (step 5) regardless of outcome. The pipeline has produced results (even if negative) that need to be reported alongside the intellectual phase results.

**Store classification:**
```yaml
implementation_phase_result:
  classification: "{FULL_SUCCESS | PARTIAL_SUCCESS | NO_PUSHES | TOTAL_FAILURE | USER_ABORT}"
  implementation_output: "{captured output contract}"
  phase_end_timestamp: "{timestamp}"
```

### 6. Proceed to Final Summary

```
═══════════════════════════════════════════════════
{if FULL_SUCCESS}
✅ IMPLEMENTATION PHASE COMPLETE — All stories pushed
{else if PARTIAL_SUCCESS}
⚠️ IMPLEMENTATION PHASE COMPLETE — Partial success
{else if NO_PUSHES}
ℹ️ IMPLEMENTATION PHASE COMPLETE — No branches pushed
{else if TOTAL_FAILURE}
❌ IMPLEMENTATION PHASE COMPLETE — All stories failed
{else}
🛑 IMPLEMENTATION PHASE — Aborted by user
{/if}
═══════════════════════════════════════════════════

Stories pushed: {pushed_count}/{implementation_count}
Branches on remote: {list or "none"}
Correction loops: {correction_loops_executed}

Loading Step 5: Final Summary...
═══════════════════════════════════════════════════
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ Implementation workflow invoked correctly via its first step
- ✅ All 14 implementation steps executed in sequence
- ✅ Only phase-gate-approved stories passed to implementation
- ✅ Output contract captured completely including test metrics
- ✅ Phase result classified accurately
- ✅ Always proceeds to final summary regardless of outcome

## FAILURE MODES

- ❌ Re-implementing implementation workflow logic instead of invoking it
- ❌ Passing the original story list instead of the phase-gate-approved list
- ❌ Skipping the implementation workflow's own readiness check
- ❌ Not capturing the output contract from step 14
- ❌ Not capturing test metrics (unique to implementation)
- ❌ Not proceeding to final summary after failure (results must be reported)
- ❌ Interfering with the implementation workflow's human approval gate (step 11)
