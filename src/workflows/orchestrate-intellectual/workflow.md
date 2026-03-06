---
name: orchestrate-intellectual
description: "Fan-out of document processing + cross-validation + corrective loop. Intellectual mode — no worktrees needed."
module: bmo
installed_path: '{project-root}/_bmad/bmo/workflows/orchestrate-intellectual'
status: implemented
firstStep: './steps/step-01-readiness-check.md'
---

# Orchestrate Intellectual Workflow

**Module:** bmo
**Status:** Implemented — 8 step files in `./steps/`
**Mode:** Intellectual (document artifacts only, no worktrees)
**Executor:** Tito 🧉 (Orchestrator Agent)

---

## Workflow Overview

**Goal:** Orchestrate parallel document processing across multiple sub-agents with cross-validation and corrective loops. Intellectual mode — no git worktrees, no code changes, no merge gates.

**Description:** The orchestrator validates story readiness and plans the fan-out (Phase 1), dispatches document-processing sub-agents in parallel to invoke BMAD workflows like create-story or advanced-elicitation (Phase 2), then collects results, runs cross-validation for coherence, and enters corrective loops as needed (Phase 3). Outputs are document artifacts (refined stories, specs, analysis docs) stored in `{output_folder}`.

**Trigger:** User selects [OI] from orchestrator menu or requests intellectual orchestration.

---

## Input Contract

```yaml
input:
  epic_path: string          # Path to epic file or epic ID
  stories: list[string]      # List of story file paths or story IDs
  mode: intellectual          # Fixed for this workflow
  parallel_limit: int        # Max concurrent sub-agents (from config)
```

---

## Configuration (from bmo/config.yaml)

```yaml
max_parallel_agents: 3
max_correction_loops: 3
sub_agent_timeout_minutes: 30
output_folder: "{project-root}/_bmad-output"
```

### Workflow-Specific Constants

```yaml
max_rerun_attempts: 2        # Step 4: max re-dispatches for hard processing failures
max_validator_retries: 2     # Step 5: max retries if cross-validator sub-agent fails
```

**Note on retry layers:** This workflow has two distinct retry mechanisms:
- **Re-run** (step 4): For hard failures — sub-agent crashed, timed out, or produced no output. Max 2 attempts.
- **Corrective loop** (step 6): For quality issues — cross-validation found contradictions/gaps. Max 3 iterations from config.
These are independent and cumulative — a story could be re-run up to 2 times AND corrected up to 3 times (5 total attempts in worst case). This is intentional.

---

## Step Files

| Step | File | Name | Phase | Executor |
|------|------|------|-------|----------|
| 1 | `step-01-readiness-check.md` | Readiness Check | Sequential (Phase 1) | Orchestrator |
| 2 | `step-02-fan-out-planning.md` | Fan-Out Planning | Sequential (Phase 1) | Orchestrator |
| 3 | `step-03-sub-agent-dispatch.md` | Sub-Agent Dispatch | Parallel (Phase 2) | Sub-Agents |
| 4 | `step-04-output-collection.md` | Output Collection | Sequential (Phase 3) | Orchestrator |
| 5 | `step-05-cross-validation.md` | Cross-Validation | Sequential (Phase 3) | Sub-Agent |
| 6 | `step-06-corrective-loop.md` | Corrective Loop | Corrective (Phase 3) | Mixed |
| 7 | `step-07-human-escalation.md` | Human Escalation | Sequential (Phase 3) | Orchestrator |
| 8 | `step-08-summary-report.md` | Summary Report | Sequential (Phase 3) | Orchestrator |

---

## Execution Phases

### Phase 1: Preparation (Steps 1-2) — SEQUENTIAL

```
Step 1: Validate stories → readiness classification
Step 2: Plan fan-out → assign workflows, group into parallel batches
```

All orchestrator-only. No sub-agents. Must complete before Phase 2.

### Phase 2: Document Processing (Step 3) — PARALLEL

```
Step 3: Dispatch document processors → one sub-agent per story
        Concurrency limited to max_parallel_agents
        Sub-agents: invoke BMAD workflows, write artifacts to output_folder
```

### Phase 3: Validation & Delivery (Steps 4-8) — SEQUENTIAL

```
Step 4: Collect outputs → verify artifacts exist, classify results
Step 5: Cross-validate → check coherence across all story artifacts
Step 6: Corrective loop → fix issues found by cross-validator (max 3 iterations)
Step 7: Human escalation → if circuit breaker exhausted, present to user
Step 8: Summary → execution report
```

---

## Key Differences from Implementation Mode

| Aspect | Implementation Mode | Intellectual Mode |
|--------|-------------------|------------------|
| Sub-agent workspace | Git worktrees | Main project dir + `{output_folder}` |
| Artifacts produced | Code + tests + commits | Documents, specs, refined stories |
| Git operations | Branch, commit, merge | None — no git involvement |
| Shared file snapshot | Required for mutation detection | Not needed — no code changes |
| Code review step | Dedicated reviewer dispatch | Not applicable |
| Merge gate | Pre-merge conflict detection | Not applicable |
| Push step | Push approved branches | Not applicable |
| Cleanup step | Remove worktrees | Not applicable |
| Cross-validation focus | Code conflicts, interfaces | Terminology, contradictions, coverage |
| BMAD workflows invoked | dev-story | create-story, advanced-elicitation, etc. |

---

## Quality Gates

| Gate | Step | Validates | Failure Action |
|------|------|-----------|----------------|
| Readiness Check | 1 | Stories have complete ACs, no ambiguities | Flag incomplete, proceed with ready |
| Post-Processing | 4 | Outputs exist and meet story ACs | Enter corrective loop |
| Post-Validation | 5 | No conflicts, no gaps across outputs | Enter corrective loop or escalate |
| Circuit Breaker | 6 | Max 3 correction attempts per story | Escalate to human (step 7) |

---

## Sub-Agent Contracts

### Document Processor (Steps 3, 6)

```yaml
role: document_processor
task_type: string           # "story_refinement" | "document_creation" | "analysis"
story_file_path: string     # Story to process
epic_context: string        # Summary of the epic
workflow_to_invoke: string  # Which BMAD workflow (e.g., "create-story")
output_path: string         # Where to write artifacts
exit_criteria:
  - "All acceptance criteria addressed in output"
  - "Output follows document standards"
  - "Artifacts written to {output_path}"
reporting: {status, artifacts_produced, summary, issues_encountered}
```

### Cross-Validator (Step 5)

```yaml
role: cross_validator
epic_file: string
artifacts: list[{story_id, artifact_path, summary}]
validation_checklist:
  - "No contradictions between stories"
  - "No duplicate requirements across stories"
  - "Shared terminology is consistent"
  - "Dependencies between stories are acknowledged"
  - "Epic goals are fully covered"
reporting: {status, conflicts, gaps, recommendations}
```

---

## Security Rules

1. Sub-agents NEVER perform git operations — intellectual mode is document-only
2. Sub-agents write artifacts ONLY to their assigned `{output_path}`
3. Sub-agents NEVER modify the original story files — only read them
4. Cross-validator is strictly READ-ONLY
5. Corrective loop re-dispatch includes FULL validator feedback

---

## Output Contract

```yaml
output:
  execution_summary:
    stories_completed: list
    stories_failed: list
    stories_pending_review: list    # Force-approved stories with known issues
    stories_excluded: list          # Stories excluded at readiness check
    stories_dropped: list           # Stories dropped at escalation
  validation_status: enum[coherent, issues_found, issues_force_accepted, skipped, skipped_no_stories]
  correction_loops_executed: int
  actions_pending_user_approval: list
  artifacts_produced:
    - story_id: string
      artifact_path: string
      artifact_type: string
```

---

## Integration Points

- **BMM**: create-story, create-epics-and-stories (invoked by document processor sub-agents)
- **Core**: advanced-elicitation (invoked by sub-agents), party-mode (for cross-validation discussions)
- **TEA**: N/A (no code in intellectual mode)

**Note:** TEA module integration is not applicable to intellectual mode.

---

## Corrective Loop with Circuit Breaker

```
max_correction_loops: 3 (from config)

LOOP per story:
  1. Cross-validator returns issues_found
  2. Orchestrator re-dispatches document processor with validator feedback
  3. Processor revises artifacts
  4. Cross-validator re-validates
  5. IF coherent → exit loop
  6. IF issues_found AND loop_count < max → repeat
  7. IF issues_found AND loop_count >= max → ESCALATE
     - Present findings and history to user
     - User decides: force approve, manual fix, or drop story
```

---

## Recovery Reference

Intellectual mode has no worktrees or branches to clean up. Recovery is limited to:

```
- Re-running failed sub-agents (step 6 corrective loop)
- Manual editing of artifacts in {output_folder}
- Dropping problematic stories and re-running
```

## Known Limitations (v1)

1. **Task tool has no timeout mechanism** — `sub_agent_timeout_minutes` is tracked for reporting but cannot be enforced. If a sub-agent hangs, manual intervention (kill session) is required.
2. **Task tool dispatch is batch-atomic** — all sub-agents in a batch complete before the orchestrator regains control. Real-time per-agent status display is post-batch only.
3. **State persistence is context-window only** — no disk persistence of orchestration state between steps. For large orchestrations (6+ stories with corrections), consider state persistence to disk (v2 enhancement).
4. **No run-id namespacing** — multiple runs for the same stories write to the same output directories. Previous run artifacts are overwritten.
5. **BMAD workflow availability** — sub-agents invoke workflows like `create-story`, `advanced-elicitation`. These must exist in the project's BMAD installation for intellectual mode to function.

---

_Spec created on 2026-03-04 via BMAD Module workflow_
_Implemented on 2026-03-05 via create-workflow [C]onvert mode_
