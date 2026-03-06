---
name: orchestrate-recovery
description: "Compensation workflow for handling sub-agent failures, orphaned worktrees, interrupted pipelines, and cleanup operations."
module: bmo
installed_path: '{project-root}/_bmad/bmo/workflows/orchestrate-recovery'
status: implemented
firstStep: './steps/step-01-scan-worktrees.md'
---

# Orchestrate Recovery Workflow

**Module:** bmo
**Status:** Implemented — 6 step files in `./steps/`
**Mode:** Utility (invoked on-demand or automatically on failure)
**Executor:** Tito 🧉 (Orchestrator Agent)

---

## Workflow Overview

**Goal:** Handle recovery scenarios from orchestration failures including sub-agent failures, orphaned worktrees, interrupted pipelines, and merge conflict resolution.

**Description:** Provides compensation actions for all failure modes in BMO orchestration. Scans worktrees and branches to reconstruct state, presents findings to user, and executes cleanup or facilitates resume based on user decision. Invoked manually via the cleanup menu item [CW].

**Trigger:** User selects [CW] from orchestrator menu.

> **v1 NOTE:** Recovery is manual invocation only. Automatic invocation from failed orchestration runs (e.g., a failsafe hook in implementation steps) is planned for v2.

---

## Input Contract

```yaml
input:
  action: enum[scan, cleanup_all, cleanup_selective, resume]
  worktree_base_path: string   # From config
  force: boolean               # Force cleanup even with uncommitted changes (default: false)
```

---

## Configuration (from bmo/config.yaml)

```yaml
max_parallel_agents: 3
max_correction_loops: 3
sub_agent_timeout_minutes: 30
worktree_base_path: "{project-root}/.claude/worktrees"
```

---

## Execution Entry Point

To execute this workflow, load and execute the first step file:
→ Load `{firstStep}` (`./steps/step-01-scan-worktrees.md`), read it completely, then follow its instructions.
→ Each step's `nextStepFile` frontmatter points to the subsequent step.
→ The workflow may exit early at step 3 (clean environment) or step 4 (resume/abort).

---

## Step Files

| Step | File | Name | Phase | Executor |
|------|------|------|-------|----------|
| 1 | `step-01-scan-worktrees.md` | Scan Worktrees | Sequential | Orchestrator |
| 2 | `step-02-scan-branches.md` | Scan Branches | Sequential | Orchestrator |
| 3 | `step-03-present-status.md` | Present Status | Sequential | Orchestrator |
| 4 | `step-04-user-decision.md` | User Decision | Sequential | Orchestrator |
| 5 | `step-05-execute-cleanup.md` | Execute Cleanup | Sequential | Orchestrator |
| 6 | `step-06-verify-clean-state.md` | Verify Clean State | Sequential | Orchestrator |

---

## Execution Flow

```
Step 1: Scan worktrees → classify active, orphaned, stale, prunable
Step 2: Scan branches → identify BMO branches, check push/commit status
Step 3: Present status → show comprehensive findings to user
Step 4: User decision → cleanup all, cleanup selective, resume, or abort
Step 5: Execute cleanup → remove worktrees, delete branches, prune refs
Step 6: Verify clean state → fresh scan to confirm cleanup succeeded
```

All steps are sequential, orchestrator-only. No sub-agents are dispatched.

**Early exits:**
- Step 3: If no BMO artifacts found → workflow ends (clean environment)
- Step 4: If user chooses [R] Resume → collect info and end (hand off to orchestrate-implementation)
- Step 4: If user chooses [X] Abort → workflow ends (no changes made)

---

## Recovery Scenarios

| Scenario | Recovery Action |
|----------|----------------|
| Sub-agent fails mid-execution | Log failure, mark story as failed, continue with others |
| Sub-agent hangs (timeout) | Kill after {sub_agent_timeout_minutes} minutes, mark as failed |
| User closes session | Worktrees persist in git, branches persist — user can resume or cleanup via menu |
| Merge conflict detected | Present conflict details to user, don't auto-resolve |
| Orphaned worktrees from previous run | Detect and remove stale worktrees |

---

## Key Design Decisions

1. **"Scan IS state reconstruction"** — Step 1 scans worktrees and deduces what happened. No state file needed for v1.
2. **"Resume = re-invoke implementation with existing worktrees"** — Not magic, just informed re-execution. Recovery collects worktree info and hands it back.
3. **Simple over complex** — This is the safety net. Fewer moving parts = fewer breakage points.
4. **All orchestrator, no sub-agents** — Recovery is too sensitive for parallel execution.

---

## Security Rules — NON-NEGOTIABLE

1. NEVER force-delete worktrees with uncommitted changes without user confirmation
2. NEVER delete branches that have been pushed to remote without user confirmation
3. Always present what will be deleted BEFORE executing
4. Log all cleanup actions for audit trail — audit output included in final step summary
5. "Cleanup All" requires typing "DELETE" to confirm
6. Safe branch deletion (`-d`) first — force (`-D`) only with explicit user confirmation
7. Validate `worktree_base_path` is inside project root before any operations
8. NEVER include the main worktree in any removal targets
9. NEVER execute `rm -rf` on paths outside `worktree_base_path`

---

## Output Contract

```yaml
output:
  worktrees_found: list[{path, branch, status}]
  worktrees_cleaned: list[string]
  branches_deleted: list[string]
  uncommitted_work_detected: boolean
  recovery_status: enum[clean, partial, user_action_needed]
```

---

## Git Commands Reference

```bash
git worktree list --porcelain  # Scan existing worktrees (detailed)
git worktree remove <path>     # Remove worktree (safe)
git worktree remove <path> --force  # Force remove worktree
git worktree prune             # Clean stale references
git branch -d <branch>         # Delete branch (safe — merged only)
git branch -D <branch>         # Force delete branch (requires user confirmation)
```

---

## Integration Points

- **orchestrate-implementation**: Recovery can hand off resume context to this workflow
- **Tito 🧉 Menu**: Invoked via [CW] Cleanup Worktrees menu option
- **Step 13 of orchestrate-implementation**: Shares cleanup patterns (worktree remove, branch delete, prune)

---

_Spec created on 2026-03-04 via BMAD Module workflow_
_Implemented on 2026-03-05 via create-workflow [C]onvert mode_
