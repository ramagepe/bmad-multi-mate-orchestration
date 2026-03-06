---
name: step-13-cleanup
description: "Remove worktrees for completed stories"
nextStepFile: './step-14-summary-report.md'
phase: sequential
phase_number: 3
executor: orchestrator
---

# Step 13: Cleanup

**Progress: Step 13 of 14** — Next: Summary Report
**Phase:** 3 (SEQUENTIAL — Orchestrator Only)

---

## STEP GOAL

Clean up git worktrees and optionally local branches for stories that have been pushed to remote. Preserve worktrees for unpushed branches so work isn't lost. This follows the orchestrate-recovery patterns.

---

## MANDATORY EXECUTION RULES

- 🛑 NEVER remove worktrees for UNPUSHED branches — work would be lost
- 📖 Always confirm with user before removing worktrees
- 🚫 Use safe branch deletion (`-d`) — never force delete merged branches
- 🎯 Leave a clean environment — no orphaned worktrees

---

## EXECUTION SEQUENCE

> **Note:** IF step 12 was skipped (cancelled by user in step 11), treat ALL worktrees as `must_preserve` (nothing was pushed — all work exists only locally).

### 1. Categorize Worktrees for Cleanup

```yaml
cleanup_plan:
  safe_to_remove:  # Pushed to remote — local copy can go
    - story_id: "{story-id-1}"
      worktree: "{path}"
      branch: "{branch_name}"
      status: "pushed"
  must_preserve:  # NOT pushed — work only exists locally
    - story_id: "{story-id-2}"
      worktree: "{path}"
      branch: "{branch_name}"
      status: "skipped"  # or "failed_push"
      reason: "Branch not pushed — local only"
  already_clean:  # Stories that failed or were dropped earlier
    - story_id: "{story-id-4}"
      status: "dropped_in_corrective_loop"
```

### 2. Present Cleanup Plan

```
🧹 CLEANUP PLAN
═══════════════════════════════════════

SAFE TO REMOVE (pushed to remote):
  🗑️ {story-id-1} — {worktree_path}
  🗑️ {story-id-3} — {worktree_path}

MUST PRESERVE (not pushed — work only exists locally):
  📌 {story-id-2} — {worktree_path}
     Reason: Branch skipped during approval
     ⚠️ To push later: git -C {worktree_path} push origin {branch_name}

{if already_clean}
ALREADY HANDLED:
  ⬜ {story-id-4} — dropped during corrective loop
{/if}

Options:
[C] Clean up pushed branches (remove worktrees + delete local branches)
[W] Remove worktrees only (keep local branches)
[K] Keep everything — skip cleanup
[A] Clean ALL (including unpushed — ⚠️ DESTRUCTIVE)
```

### 3. Execute Cleanup

**For each worktree being removed:**

```bash
# 1. Remove the worktree
git worktree remove {worktree_path}

# Verify removal
ls {worktree_path} 2>/dev/null && echo "STILL EXISTS" || echo "REMOVED"

# 2. Optionally delete the local branch (if user chose [C])
git branch -d {branch_name}
# -d is safe: only deletes if branch is fully merged to its upstream
# If it fails, the branch has unmerged changes — keep it

# 3. Prune stale worktree references
git worktree prune
```

**For the [A] (clean all) option — extra safety:**

```
⚠️ WARNING: You are about to remove worktrees for UNPUSHED branches.
This work exists ONLY locally and will be LOST.

Branches that will be lost:
  • {story-id-2}: {commit_count} commits, {files_count} files changed

Type "DELETE" to confirm, or anything else to cancel:
```

### 4. Handle Cleanup Errors

If a worktree removal fails:

```bash
# Force removal if normal removal fails
git worktree remove {worktree_path} --force

# If that also fails, manual cleanup
rm -rf {worktree_path}
git worktree prune
```

Report any errors to user but continue with other cleanups.

### 5. Verify Clean State

```bash
# List remaining worktrees
git worktree list

# List remaining local branches related to this workflow
git branch --list | grep -E "{story-id-pattern}"
```

### 6. Present Cleanup Results

```
🧹 CLEANUP COMPLETE
═══════════════════════════════════════

Removed: {removed_count} worktrees
  ✅ {story-id-1} — worktree removed, branch deleted
  ✅ {story-id-3} — worktree removed, branch deleted

Preserved: {preserved_count} worktrees
  📌 {story-id-2} — {worktree_path} (not pushed)

Remaining worktrees:
{git worktree list output}
```

### 7. Store Cleanup State

```yaml
cleanup_state:
  worktrees_removed: list[string]
  branches_deleted: list[string]
  worktrees_preserved: list[{story_id, path, reason}]
  cleanup_errors: list[string]
```

### 8. Proceed to Summary

```
✅ Cleanup complete.
   Loading Step 14: Summary Report...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ Pushed branches cleaned up (worktree + branch)
- ✅ Unpushed branches preserved with clear instructions
- ✅ No orphaned worktrees left
- ✅ User informed of what was cleaned and what remains

## FAILURE MODES

- ❌ **CRITICAL**: Removing worktrees for unpushed branches without confirmation
- ❌ Force-deleting branches that have unmerged work
- ❌ Leaving orphaned worktree references
- ❌ Not reporting preserved worktrees and how to use them later
