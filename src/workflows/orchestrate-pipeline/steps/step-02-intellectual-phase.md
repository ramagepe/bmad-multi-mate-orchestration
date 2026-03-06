---
name: step-02-intellectual-phase
description: "Invoke orchestrate-intellectual workflow for document refinement"
nextStepFile: './step-03-phase-gate.md'
phase: sequential
phase_number: 1
executor: orchestrator
---

# Step 2: Intellectual Phase

**Progress: Step 2 of 5** — Next: Phase Gate
**Phase:** 1 (SEQUENTIAL — Sub-Workflow Invocation)

---

## STEP GOAL

Invoke the orchestrate-intellectual workflow to run the complete document refinement pipeline. This step does NOT re-implement the intellectual workflow logic — it delegates to the existing 8-step workflow and captures the output contract when complete. The orchestrator manages the lifecycle: start, monitor for completion or failure, and capture results.

---

## MANDATORY EXECUTION RULES

- 🛑 NEVER re-implement intellectual workflow steps — invoke the existing workflow
- 📖 Load and execute the intellectual workflow's first step file directly
- 🚫 NEVER skip the intellectual workflow's own readiness check (its step 1)
- 🎯 Capture the COMPLETE output contract from the intellectual workflow
- ⚠️ If the intellectual workflow fails or is aborted, handle gracefully — do NOT proceed to implementation

---

## EXECUTION SEQUENCE

### 1. Announce Phase Start

```
═══════════════════════════════════════════════════
🧠 PHASE 1: INTELLECTUAL — Document Refinement
═══════════════════════════════════════════════════

Invoking: orchestrate-intellectual workflow
Path: {intellectual_workflow_path}
Stories: {story_count} stories
Mode: intellectual

This phase will:
  • Validate story readiness (intellectual step 1)
  • Plan document processing fan-out (intellectual step 2)
  • Dispatch document processor sub-agents (intellectual step 3)
  • Collect and validate outputs (intellectual steps 4-5)
  • Run corrective loops if needed (intellectual step 6)
  • Escalate unresolved issues (intellectual step 7)
  • Generate intellectual summary (intellectual step 8)

Starting intellectual workflow...
───────────────────────────────────────────────────
```

### 2. Prepare Intellectual Workflow Inputs

Map pipeline state to intellectual workflow input contract:

```yaml
intellectual_input:
  epic_path: "{pipeline_state.epic_path}"
  stories: "{pipeline_state.stories}"
  mode: intellectual
  parallel_limit: "{pipeline_state.parallel_limit}"
```

**Note:** The intellectual workflow will load its own config values (output_folder, max_correction_loops, etc.) from bmo/config.yaml. The pipeline does not need to pass these.

### 3. Execute Intellectual Workflow

Load and execute the intellectual workflow's first step:

```
Load file: {intellectual_workflow_path}/steps/step-01-readiness-check.md
```

**CRITICAL:** Execute the intellectual workflow's 8 steps IN SEQUENCE, following each step's `nextStepFile` chain:

```
step-01-readiness-check.md
  → step-02-fan-out-planning.md
    → step-03-sub-agent-dispatch.md
      → step-04-output-collection.md
        → step-05-cross-validation.md
          → step-06-corrective-loop.md
            → step-07-human-escalation.md
              → step-08-summary-report.md (final_step: true — RETURN TO PIPELINE)
```

Each step file is self-contained with its own execution sequence. Follow them exactly as written.

**⚠️ RETURN TO PIPELINE:** When `step-08-summary-report.md` completes (it is marked `final_step: true` and has NO `nextStepFile`), you MUST return here to this pipeline step and continue at sub-step 4 below ("Capture Intellectual Output Contract"). Do NOT stop execution — the sub-workflow is complete but the pipeline is not.

**Important behavioral notes:**
- The intellectual workflow's step 1 will perform its OWN readiness check — allow it. It may exclude stories that passed our lighter check.
- The intellectual workflow's steps 6-7 may involve user interaction (escalation decisions) — this is expected.
- The intellectual workflow's step 8 generates a summary report — this is the output we need to capture.

### 4. Capture Intellectual Output Contract

After the intellectual workflow completes (step 8 summary report), capture its output contract:

```yaml
intellectual_output:
  execution_summary:
    stories_completed: list        # Stories with approved artifacts
    stories_failed: list           # Stories that failed processing
    stories_pending_review: list   # Force-approved stories with known issues
    stories_excluded: list         # Stories excluded at readiness check
    stories_dropped: list          # Stories dropped at escalation
  validation_status: string        # coherent | issues_found | issues_force_accepted | skipped | skipped_no_stories
  correction_loops_executed: int
  actions_pending_user_approval: list
  artifacts_produced:
    - story_id: string
      artifact_path: string
      artifact_type: string
```

**⚠️ Field name mapping:** The intellectual workflow uses `actions_pending_user_approval` in its output contract. The pipeline uses `actions_pending_approval`. Map this field when capturing: `actions_pending_approval = intellectual_output.actions_pending_user_approval`.

Store this output — it will be:
- Presented to the user in step 3 (phase gate)
- Included in the final summary report (step 5)

### 5. Classify Intellectual Phase Result

Based on the output contract, classify the phase result:

| Condition | Classification | Pipeline Action |
|-----------|---------------|-----------------|
| All stories completed, validation coherent | ✅ FULL_SUCCESS | Proceed to phase gate |
| Most stories completed, minor issues | ⚠️ PARTIAL_SUCCESS | Proceed to phase gate |
| All stories failed or dropped | ❌ TOTAL_FAILURE | Present to user, recommend abort |
| No stories processed (all excluded) | ❌ NO_STORIES | Present to user, recommend abort |
| User aborted during intellectual workflow | 🛑 USER_ABORT | Respect abort, stop pipeline |

**Store classification:**
```yaml
intellectual_phase_result:
  classification: "{FULL_SUCCESS | PARTIAL_SUCCESS | TOTAL_FAILURE | NO_STORIES | USER_ABORT}"
  stories_available_for_implementation: list  # completed + pending_review (not failed/dropped/excluded)
  intellectual_output: "{captured output contract}"
  phase_end_timestamp: "{timestamp}"
```

### 6. Handle Phase Failure

**IF classification is TOTAL_FAILURE or NO_STORIES:**

```
═══════════════════════════════════════════════════
🛑 INTELLECTUAL PHASE — {TOTAL_FAILURE | NO_STORIES}
═══════════════════════════════════════════════════

{if TOTAL_FAILURE}
All stories failed during the intellectual phase.
No refined documents are available for implementation.

Failed stories:
  {list each failed/dropped story with reason}
{/if}

{if NO_STORIES}
No stories were processed — all were excluded at readiness check.

Excluded stories:
  {list each excluded story with reason}
{/if}

───────────────────────────────────────────────────

Options:
[R] Re-run intellectual phase (same inputs)
[A] Abort pipeline
```

- **R**: Return to the beginning of this step (step 2, sub-step 1)
- **A**: STOP pipeline. Report what was attempted and why it failed.

**IF classification is USER_ABORT:**

```
ℹ️ Intellectual phase was aborted by user.
   Pipeline stopped. No implementation phase will run.
```

STOP pipeline.

### 7. Proceed to Phase Gate

**IF classification is FULL_SUCCESS or PARTIAL_SUCCESS:**

```
═══════════════════════════════════════════════════
✅ INTELLECTUAL PHASE COMPLETE
═══════════════════════════════════════════════════

Result: {classification}
Stories completed: {completed_count}
Stories available for implementation: {available_count}
Artifacts produced: {artifact_count}

Loading Step 3: Phase Gate — Intellectual → Implementation...
═══════════════════════════════════════════════════
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ Intellectual workflow invoked correctly via its first step
- ✅ All 8 intellectual steps executed in sequence
- ✅ Output contract captured completely
- ✅ Phase result classified accurately
- ✅ Failure cases handled with user options
- ✅ Stories available for implementation identified

## FAILURE MODES

- ❌ Re-implementing intellectual workflow logic instead of invoking it
- ❌ Skipping the intellectual workflow's own readiness check
- ❌ Not capturing the output contract from step 8
- ❌ Auto-proceeding to implementation after total failure
- ❌ Not tracking which stories are available for implementation
- ❌ Not handling user abort during the intellectual workflow
