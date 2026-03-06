---
name: step-06-verify-clean-state
description: "Verify no orphaned worktrees or branches remain after cleanup"
phase: sequential
phase_number: 1
executor: orchestrator
final_step: true
---

# Step 6: Verify Clean State

**Progress: Step 6 of 6** — FINAL STEP
**Phase:** Sequential — Orchestrator Only

---

## STEP GOAL

Verify that cleanup was successful and no orphaned BMO resources remain. Run a fresh scan to confirm the environment is clean. Produce the final recovery output contract. This is the safety net's safety net — trust but verify.

---

## MANDATORY EXECUTION RULES

- 🛑 Run a FRESH scan — do NOT rely on cleanup results from step 5 alone
- 📖 Compare current state against expected state after cleanup
- 🚫 NEVER report "clean" if orphaned resources are found
- 🎯 Produce the complete output contract for the recovery workflow

---

## EXECUTION SEQUENCE

### 1. Run Fresh Worktree Scan

Re-scan the environment from scratch (same commands as step 1):

```bash
# Full worktree listing
git worktree list --porcelain

# Check worktree base directory
ls -la {worktree_base_path} 2>/dev/null || echo "DIRECTORY_NOT_FOUND"

# Check for any remaining BMO directories
ls -1d {worktree_base_path}/*/ 2>/dev/null || echo "NO_SUBDIRECTORIES"
```

### 2. Run Fresh Branch Scan

```bash
# List remaining local branches
git branch --list --format='%(refname:short) %(objectname:short) %(upstream:track)'

# Check for stale worktree references
# NOTE: `git worktree list --porcelain` does NOT emit "prunable" — use prune --dry-run
git worktree prune --dry-run 2>&1 || echo "NO_PRUNABLE"
```

### 3. Compare Against Expected State

Build comparison:

```yaml
verification:
  expected_removed:
    worktrees: list[string]  # From step 5 cleanup_results
    branches: list[string]   # From step 5 cleanup_results
  expected_preserved:
    worktrees: list[string]  # Items user chose to keep
    branches: list[string]   # Branches preserved (unmerged)
  actual_remaining:
    worktrees: list[string]  # From fresh scan
    branches: list[string]   # From fresh scan
```

**Check for discrepancies:**

| Expected | Actual | Status |
|----------|--------|--------|
| Removed | Gone | ✅ Verified |
| Removed | Still exists | ❌ Cleanup failed |
| Preserved | Still exists | ✅ Correct |
| Preserved | Gone | ⚠️ Unexpected removal |
| Not targeted | Still exists | ℹ️ Unrelated, expected |

### 4. Handle Remaining Issues

**IF orphaned worktrees still exist after cleanup:**

```
⚠️ VERIFICATION: Residual worktrees detected
═══════════════════════════════════════════════════

The following worktrees were targeted for removal but still exist:

{for each residual:}
  ❌ {worktree_path}
     Attempted removal: {method used}
     Error: {error from step 5}
{/for}

These may require manual cleanup:
{for each residual:}
  rm -rf {worktree_path}
{/for}
Then run: git worktree prune
```

**IF stale branch references remain:**

```
⚠️ VERIFICATION: Residual branches detected
═══════════════════════════════════════════════════

The following branches were targeted for deletion but still exist:

{for each residual:}
  ❌ {branch_name}
     Reason: {likely unmerged changes}
     To force delete: git branch -D {branch_name}
{/for}
```

### 5. Determine Recovery Status

```yaml
recovery_status_determination:
  if: all_targeted_removed AND no_unexpected_removals
  then: "clean"

  if: some_targeted_removed AND some_remaining
  then: "partial"

  if: uncommitted_work_preserved OR user_action_needed
  then: "user_action_needed"
```

### 6. Produce Final Output Contract

Compile the complete recovery output:

```yaml
output:
  worktrees_found:
    - path: "{path}"
      branch: "{branch}"
      status: "{active|orphaned|stale|removed|preserved}"
    # ... for all worktrees discovered in scan
  worktrees_cleaned: list[string]       # Paths of successfully removed worktrees
  branches_deleted: list[string]        # Names of successfully deleted branches
  uncommitted_work_detected: boolean    # From step 2 scan
  recovery_status: "{clean|partial|user_action_needed}"
```

### 7. Present Final Verification Report

```
🔍 VERIFICATION REPORT
═══════════════════════════════════════════════════

Recovery Status: {clean ✅ | partial ⚠️ | user_action_needed ❌}

{if status == clean}
✅ CLEAN — All targeted resources successfully removed.

  Worktrees removed: {count}
  Branches deleted: {count}
  References pruned: {count}
  
  Remaining worktrees (expected):
  {git worktree list output — should only show main worktree + preserved items}

  Environment is clean. No further action needed.

{elif status == partial}
⚠️ PARTIAL — Some cleanup operations failed.

  Successfully removed: {success_count}
  Failed to remove: {failed_count}
  
  Residual items requiring manual cleanup:
  {list residual items with manual cleanup commands}
  
  Preserved items (as requested):
  {list preserved items}

{elif status == user_action_needed}
❌ USER ACTION NEEDED — Manual intervention required.

  {describe what needs manual attention}
  
  Recommended actions:
  {list specific commands or steps}
{/if}
```

### 8. Cleanup Audit Summary

```
📋 CLEANUP AUDIT TRAIL
═══════════════════════════════════════════════════

Total operations: {count}
Successful: {count}
Failed: {count}

{for each cleanup_log_entry:}
  [{timestamp}] {action} {target} — {result} (method: {method})
{/for}
```

### 9. Final State and Workflow End

```yaml
recovery_state:
  # Complete final state
  scan_phase: "verified"
  recovery_status: "{clean|partial|user_action_needed}"
  output: {complete output contract}
  audit_trail: list[cleanup_log_entry]
```

```
═══════════════════════════════════════════════════
🏁 BMO RECOVERY COMPLETE
═══════════════════════════════════════════════════

Recovery Status: {status}
Worktrees cleaned: {count}
Branches deleted: {count}

{if preserved_items}
📌 Preserved items:
{for each preserved:}
  • {item} — {reason}
{/for}
{/if}

{if status != clean}
📋 Follow-up actions listed above.
{/if}

Recovery workflow finished. 🧉
═══════════════════════════════════════════════════
```

---

## OUTPUT CONTRACT FULFILLMENT

This step produces the final output matching the workflow's output contract:

```yaml
output:
  worktrees_found: list[{path, branch, status}]
  worktrees_cleaned: list[string]
  branches_deleted: list[string]
  uncommitted_work_detected: boolean
  recovery_status: enum[clean, partial, user_action_needed]
```

---

## SUCCESS METRICS

- ✅ Fresh scan performed (not relying solely on step 5 results)
- ✅ All targeted removals verified as complete
- ✅ Preserved items confirmed still intact
- ✅ Discrepancies clearly reported with manual fix commands
- ✅ Complete output contract produced
- ✅ Full audit trail available

## FAILURE MODES

- ❌ Reporting "clean" when orphaned resources remain
- ❌ Not running a fresh scan (trusting step 5 blindly)
- ❌ Not providing manual cleanup commands for residual items
- ❌ Missing items in the output contract
- ❌ Not producing the audit trail
