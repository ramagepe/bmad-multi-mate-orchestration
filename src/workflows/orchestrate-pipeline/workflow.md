---
name: orchestrate-pipeline
description: "Composition of intellectual + implementation in a single flow. Mixed mode — refinement then implementation then validation."
module: bmo
installed_path: '{project-root}/_bmad/bmo/workflows/orchestrate-pipeline'
status: implemented
firstStep: './steps/step-01-pipeline-readiness.md'
---

# Orchestrate Full Pipeline Workflow

**Module:** bmo
**Status:** Implemented — 5 step files in `./steps/`
**Mode:** Mixed (intellectual phase → implementation phase)
**Executor:** Tito 🧉 (Orchestrator Agent)

---

## Workflow Overview

**Goal:** Compose the intellectual refinement and implementation orchestration workflows into a single end-to-end pipeline with quality gates between phases.

**Description:** Runs orchestrate-intellectual first (document refinement, cross-validation), then transitions to orchestrate-implementation (worktree-isolated parallel dev, code review, merge gate). A human approval gate separates the phases — the user reviews intellectual results and explicitly approves the transition to implementation. Worktrees are only created during the implementation phase.

**Trigger:** User selects [OP] from orchestrator menu or requests full pipeline orchestration.

---

## Input Contract

```yaml
input:
  epic_path: string          # Path to epic file or epic ID
  stories: list[string]      # List of story file paths or story IDs
  mode: mixed                 # Fixed for this workflow
  parallel_limit: int        # Max concurrent sub-agents (from config)
  base_branch: string        # Branch to create worktrees from (used in implementation phase)
```

---

## Configuration (from bmo/config.yaml)

```yaml
max_parallel_agents: 3
max_correction_loops: 3
sub_agent_timeout_minutes: 30
worktree_base_path: "{project-root}/.claude/worktrees"
output_folder: "{project-root}/_bmad-output"
```

**Note:** Configuration values are consumed by the sub-workflows, not the pipeline directly. The pipeline reads config only for parallel_limit and output_folder during readiness check.

---

## Step Files

| Step | File | Name | Phase | Executor |
|------|------|------|-------|----------|
| 1 | `step-01-pipeline-readiness.md` | Pipeline Readiness Check | Sequential (Phase 1) | Orchestrator |
| 2 | `step-02-intellectual-phase.md` | Intellectual Phase | Sequential (Phase 1) | Orchestrator → Sub-Workflow |
| 3 | `step-03-phase-gate.md` | Phase Gate: Intellectual → Implementation | Sequential (Phase 2) | Orchestrator |
| 4 | `step-04-implementation-phase.md` | Implementation Phase | Sequential (Phase 3) | Orchestrator → Sub-Workflow |
| 5 | `step-05-final-summary.md` | Final Summary | Sequential (Phase 3) | Orchestrator |

---

## Execution Phases

### Phase 1: Intellectual (Steps 1-2) — SEQUENTIAL

```
Step 1: Pipeline readiness → validate epic, stories, sub-workflows, collect base_branch
Step 2: Invoke orchestrate-intellectual → 8-step sub-workflow runs end-to-end
         (readiness → fan-out → dispatch → collect → validate → correct → escalate → summarize)
```

Pipeline manages lifecycle. Sub-workflow executes its own 8 steps internally.

### Phase 2: Phase Transition (Step 3) — HUMAN DECISION

```
Step 3: Phase gate → present intellectual results, get user approval
         Options: proceed / select stories / re-run intellectual / abort
         CRITICAL: user MUST explicitly approve before implementation starts
```

### Phase 3: Implementation + Reporting (Steps 4-5) — SEQUENTIAL

```
Step 4: Invoke orchestrate-implementation → 14-step sub-workflow runs end-to-end
         (readiness → snapshot → worktrees → fanout → collect → review → correct →
          validate → mutations → pre-merge → approval → push → cleanup → summarize)
Step 5: Combined summary → merged report from both phases
```

Pipeline manages lifecycle. Sub-workflow executes its own 14 steps internally (including its own human approval gate for branch pushes at step 11).

---

## Quality Gates

| Gate | Step | Validates | Failure Action |
|------|------|-----------|----------------|
| Pipeline Readiness | 1 | Epic, stories, sub-workflows exist | Abort with details |
| Phase Transition | 3 | Intellectual phase completed with usable results | Halt, present issues, user decides |
| All intellectual gates | 2 (internal) | (inherited from orchestrate-intellectual) | (inherited — handled by sub-workflow) |
| All implementation gates | 4 (internal) | (inherited from orchestrate-implementation) | (inherited — handled by sub-workflow) |

---

## Sub-Workflow Composition

This is a **composition workflow** — it does not dispatch sub-agents directly. Instead it invokes two complete sub-workflows sequentially:

### orchestrate-intellectual (Phase 1)

```yaml
invoked_at: step-02-intellectual-phase.md
workflow_path: "{project-root}/_bmad/bmo/workflows/orchestrate-intellectual/workflow.md"
steps: 8 (readiness → fan-out → dispatch → collect → validate → correct → escalate → summarize)
mode: intellectual
produces: refined document artifacts, cross-validation report
input_from_pipeline: epic_path, stories, parallel_limit
output_captured: execution_summary, validation_status, artifacts_produced
```

### orchestrate-implementation (Phase 2)

```yaml
invoked_at: step-04-implementation-phase.md
workflow_path: "{project-root}/_bmad/bmo/workflows/orchestrate-implementation/workflow.md"
steps: 14 (readiness → snapshot → worktrees → fanout → collect → review → correct → validate → mutations → pre-merge → approval → push → cleanup → summarize)
mode: implementation
produces: implemented code branches, test results, push confirmations
input_from_pipeline: epic_path, stories_for_implementation, parallel_limit, base_branch
output_captured: execution_summary, branches_pushed, test_metrics, shared_file_mutations
```

**Important:** The story list passed to implementation may be a SUBSET of the original — only stories that passed intellectual processing AND were approved at the phase gate.

---

## Path Resolution Convention

`nextStepFile` paths are **always resolved relative to the directory containing the file that declares them**. When the pipeline loads a sub-workflow's step file, any `nextStepFile` in that step resolves against that step's own directory — not the pipeline's. For example: pipeline step 2 loads `orchestrate-intellectual/steps/step-01-readiness-check.md`, and that file declares `nextStepFile: './step-02-fan-out-planning.md'` — this resolves to `orchestrate-intellectual/steps/step-02-fan-out-planning.md`. When a sub-workflow's final step has `final_step: true` and no `nextStepFile`, control returns to the invoking pipeline step that initiated the sub-workflow.

---

## Output Contract

```yaml
output:
  execution_summary:
    intellectual_phase:
      stories_refined: list
      stories_excluded: list          # Excluded at intellectual readiness
      stories_dropped: list           # Dropped at intellectual escalation
      cross_validation_status: string
      artifacts_produced: list        # Document artifacts from intellectual phase
    implementation_phase:
      stories_completed: list
      stories_failed: list
      stories_excluded: list          # Excluded at implementation readiness
      stories_dropped: list           # Dropped at implementation escalation
      branches_created: list[string]
      branches_pushed: list[string]   # Branches successfully pushed to remote
    stories_skipped_at_gate: list     # Stories user chose to skip at phase gate
  actions_pending_approval: list
  shared_file_mutations: list
  total_correction_loops: int
  test_metrics:                       # From implementation phase
    total_verifications: int
    independently_passed: int
    discrepancies_detected: int
    pre_merge_regressions: int
```

---

## Integration

- **orchestrate-intellectual**: Invoked as Phase 1
- **orchestrate-implementation**: Invoked as Phase 2
- All BMM, Core, and TEA integrations inherited from sub-workflows
- **BMM**: create-story, dev-story, code-review (via sub-workflows)
- **Core**: Task tool (sub-agent dispatch, via sub-workflows), advanced-elicitation (via intellectual)
- **TEA**: testarch-automate, testarch-test-review (via implementation, v2)

---

## Security Rules

1. Pipeline NEVER dispatches sub-agents directly — sub-workflows handle all dispatch
2. Phase transition requires EXPLICIT user approval (step 3, `security: critical`)
3. Implementation branch pushes require EXPLICIT user approval (inherited from implementation step 11)
4. Pipeline NEVER performs git operations directly — sub-workflows handle all git
5. Stories cannot enter implementation without passing the phase gate

---

## Known Limitations (v1)

1. **Sequential phase execution only** — intellectual phase must fully complete before implementation can start. There is no overlapping or streaming of stories between phases.
2. **No incremental re-entry** — if the pipeline is interrupted, there is no mechanism to resume from a specific phase/step. The entire pipeline must be re-run (or individual sub-workflows run via [OI]/[OD]).
3. **Story list attrition** — stories can be filtered at three points (pipeline readiness, intellectual readiness, phase gate) plus a fourth (implementation readiness). The implementation phase may receive significantly fewer stories than originally provided.
4. **Duplicate readiness checks** — the pipeline does a light readiness check (step 1), and each sub-workflow does its own detailed check (intellectual step 1, implementation step 1). This is intentional but means stories are validated three times.
5. **All sub-workflow limitations inherited** — timeout constraints, batch-atomic dispatch, state persistence, and run-id namespacing limitations from both sub-workflows apply to the pipeline.
6. **Practical story limit: 3-4 stories recommended** — the pipeline executes 5 pipeline steps + 8 intellectual steps + 14 implementation steps = 27 total step files in a single context window, plus sub-agent dispatches, user interactions, and correction loops. For 5+ stories, context window pressure becomes significant. **For 5+ stories, run phases independently via [OI] and [OD].**
7. **Phase gate re-run is limited to 2 attempts** — to prevent unbounded context consumption, the user can re-run the intellectual phase from the phase gate at most 2 times. After that, the pipeline must be aborted and restarted.
8. **Sub-workflow return is implicit** — when a sub-workflow's final step completes (final_step: true), the orchestrator must return to the pipeline step that invoked it. This relies on the orchestrator maintaining pipeline context across sub-workflow execution. State checkpoints at phase boundaries mitigate this risk.
9. **Field name mapping** — the intellectual sub-workflow uses `actions_pending_user_approval` while the pipeline uses `actions_pending_approval`. The orchestrator must map this field name when capturing intellectual output.

---

_Spec created on 2026-03-04 via BMAD Module workflow_
_Implemented on 2026-03-05 via create-workflow [C]onvert mode_
