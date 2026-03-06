---
name: step-05-execute-cleanup
description: "Execute cleanup of selected worktrees and branches with safety checks"
nextStepFile: './step-06-verify-clean-state.md'
phase: sequential
phase_number: 1
executor: orchestrator
security: critical
---

# Step 5: Execute Cleanup

**Progress: Step 5 of 6** — Next: Verify Clean State
**Phase:** Sequential — Orchestrator Only
**Security Level:** CRITICAL — Destructive operations with audit trail

---

## STEP GOAL

Execute the cleanup plan approved by the user in step 4. Remove selected worktrees, delete local branches, and prune stale references. Every action is logged for audit trail. Safety checks are enforced at each step — no data is destroyed without prior user confirmation (obtained in step 4).

---

## MANDATORY EXECUTION RULES

- 🛑 NEVER remove a worktree with uncommitted changes unless user confirmed with "DELETE" in step 4
- 📖 Verify the decision state from step 4 before executing ANY cleanup
- 🚫 NEVER use `git branch -D` unless the branch is in `requires_force_delete` list AND user confirmed
- 🎯 Log EVERY action — success and failure — for audit trail
- ⚠️ Use safe deletion first (`git worktree remove`, `git branch -d`), escalate only on failure
- 🔒 Present what will be deleted BEFORE each operation (final safety net)

---

## SECURITY RULES — NON-NEGOTIABLE

```
1. NEVER force-delete worktrees with uncommitted changes without prior user confirmation
2. NEVER delete branches pushed to remote without prior user confirmation
3. Always use safe branch delete (-d) first — only use -D if user explicitly confirmed
4. Log all cleanup actions with timestamps
5. Verify removal after each operation
6. If any operation fails unexpectedly, STOP and report to user
```

---

## EXECUTION SEQUENCE

### 1. Load and Validate Decision State

Retrieve `{recovery_state.decision}` from step 4:

```yaml
expected:
  action: "{cleanup_all|cleanup_selective}"
  targets:
    worktrees_to_remove: list[string]
    branches_to_delete: list[string]
    items_to_preserve: list[string]
  force: boolean
  requires_force_delete: list[string]
```

**Validation:**
- Confirm `action` is `cleanup_all` or `cleanup_selective` (not `resume` or `abort`)
- Confirm `targets` is non-empty
- IF `force == true`, confirm the user typed "DELETE" in step 4

### 2. Present Final Cleanup Plan

One last look before executing:

```
🧹 EXECUTING CLEANUP PLAN
═══════════════════════════════════════════════════

Worktrees to remove: {count}
{for each worktree_to_remove:}
  🗑️ {worktree_path} (branch: {branch_name})
{/for}

Branches to delete: {count}
{for each branch_to_delete:}
  🗑️ {branch_name} {if force_delete}(force){/if}
{/for}

{if items_to_preserve}
Preserving: {count}
{for each item_to_preserve:}
  📌 {item_path} — {reason}
{/for}
{/if}

Executing...
```

### 3. Remove Worktrees — One at a Time

For EACH worktree in `{worktrees_to_remove}`, execute sequentially:

**Step 3a: Safety guards — BEFORE any removal**

```
🛑 PRE-REMOVAL VALIDATION (NON-NEGOTIABLE):
1. IF worktree_path == recovery_state.worktree_scan.main_worktree → SKIP and ALERT user
   "⚠️ BLOCKED: Cannot remove main worktree {worktree_path}"
   NEVER include the main worktree in any removal targets.

2. VALIDATE that worktree_path starts with {worktree_base_path}:
   IF NOT → SKIP and ALERT user
   "⚠️ BLOCKED: Path {worktree_path} is outside worktree base directory"

3. VERIFY the worktree still exists on filesystem (guard against state drift):
   ls {worktree_path} 2>/dev/null || { log "already_removed" and SKIP }
```

**Step 3b: Safe removal attempt**

```bash
# Attempt safe worktree removal
git worktree remove "{worktree_path}"
```

**Step 3c: Verify removal**

```bash
# Verify the worktree directory is gone
ls "{worktree_path}" 2>/dev/null && echo "STILL_EXISTS" || echo "REMOVED"
```

**Step 3d: Handle failure — escalate with force gate**

IF safe removal fails:

```bash
# Try force removal (only if user confirmed force in step 4)
git worktree remove "{worktree_path}" --force
```

IF force removal also fails:

```
🛑 FORCE GATE CHECK:
IF recovery_state.decision.force == true:
  # User confirmed destructive cleanup with "DELETE" in step 4
  # Last resort: manual directory removal + prune
  # MANDATORY PATH VALIDATION before rm -rf:
  #   1. worktree_path MUST start with worktree_base_path
  #   2. worktree_path MUST NOT equal worktree_base_path
  #   3. worktree_path MUST contain at least 4 path segments
  #   4. worktree_path MUST NOT be /, /home, /tmp, /usr, /etc, or project root
  rm -rf "{worktree_path}"
  git worktree prune
ELSE:
  # STOP — cannot remove without force confirmation
  # Do NOT escalate to rm -rf without user having typed "DELETE"
  Log failure: "worktree_removal_blocked — force not authorized"
  Add to worktrees_failed list with reason "requires force confirmation"
  Continue with next worktree
```

**Step 3e: Verify after escalation**

```bash
# Final verification
ls "{worktree_path}" 2>/dev/null && echo "STILL_EXISTS" || echo "REMOVED"
```

**Log each operation:**

```yaml
cleanup_log_entry:
  timestamp: "{timestamp}"
  action: "worktree_remove"
  target: "{worktree_path}"
  method: "{safe|force|manual}"
  result: "{success|failed}"
  error: "{error_message if failed}"
```

**Report progress after each worktree:**

```
  {✅|❌} {worktree_path} — {removed|FAILED: reason}
```

### 4. Delete Local Branches

After ALL worktrees are removed, delete associated branches:

**For branches with safe delete:**

```bash
# Safe delete — only succeeds if branch is merged to upstream
git branch -d {branch_name}
```

**For branches requiring force delete** (user confirmed in step 4):

```bash
# Force delete — user explicitly confirmed data loss
git branch -D {branch_name}
```

**Handle branch delete failures:**

```bash
# If -d fails, branch has unmerged changes
# Only use -D if branch is in requires_force_delete list
git branch -d {branch_name} 2>&1
# IF exit code != 0 AND branch NOT in requires_force_delete:
#   Report to user — branch preserved (has unmerged work)
# IF exit code != 0 AND branch IN requires_force_delete:
#   Use git branch -D {branch_name}
```

**Log each branch deletion:**

```yaml
cleanup_log_entry:
  timestamp: "{timestamp}"
  action: "branch_delete"
  target: "{branch_name}"
  method: "{safe_d|force_D}"
  result: "{success|failed|preserved}"
  error: "{error_message if failed}"
```

### 5. Prune Stale Worktree References

```bash
# Clean up any stale worktree references
git worktree prune

# Verify prune result
git worktree list --porcelain | grep -c "prunable" || echo "0"
```

### 6. Clean Up Empty Base Directory

```bash
# If worktree base directory is now empty, remove it
if [ -d "{worktree_base_path}" ] && [ -z "$(ls -A {worktree_base_path})" ]; then
    rmdir {worktree_base_path}
    echo "BASEDIR_REMOVED"
else
    echo "BASEDIR_KEPT"
fi
```

### 7. Compile Cleanup Results

```yaml
cleanup_results:
  worktrees_removed: list[string]
  worktrees_failed: list[{path, reason}]
  branches_deleted: list[string]
  branches_preserved: list[{name, reason}]
  branches_failed: list[{name, reason}]
  pruned_references: int
  basedir_removed: boolean
  total_operations: int
  successful_operations: int
  failed_operations: int
```

### 8. Present Cleanup Results

```
🧹 CLEANUP RESULTS
═══════════════════════════════════════════════════

Worktrees:
  Removed: {removed_count}
{for each removed worktree:}
    ✅ {worktree_path}
{/for}
{if worktrees_failed}
  Failed: {failed_count}
{for each failed:}
    ❌ {worktree_path} — {reason}
{/for}
{/if}

Branches:
  Deleted: {deleted_count}
{for each deleted branch:}
    ✅ {branch_name}
{/for}
{if branches_preserved}
  Preserved (unmerged work): {preserved_count}
{for each preserved:}
    📌 {branch_name} — {reason}
{/for}
{/if}

Stale references pruned: {pruned_count}
{if basedir_removed}
Base directory removed: {worktree_base_path}
{/if}

Result: {successful_operations}/{total_operations} operations succeeded
```

### 9. Store Cleanup State

```yaml
recovery_state:
  # ... previous state preserved ...
  cleanup_results: {from above}
  cleanup_log: list[cleanup_log_entry]  # Full audit trail
  scan_phase: "cleanup_executed"
```

### 10. Proceed to Verification

```
✅ Cleanup execution complete.
   {successful_operations}/{total_operations} operations succeeded.
   Loading Step 6: Verify Clean State...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ All targeted worktrees removed (or failures clearly reported)
- ✅ All targeted branches deleted with appropriate method (-d or -D)
- ✅ Stale references pruned
- ✅ Preserved items left untouched
- ✅ Every operation logged with timestamp and result
- ✅ User informed of results including any failures

## FAILURE MODES

- ❌ **CRITICAL**: Deleting worktrees with uncommitted work without prior confirmation
- ❌ **CRITICAL**: Using `git branch -D` on a branch not in `requires_force_delete` list
- ❌ Not verifying removal after each operation
- ❌ Silently ignoring failed operations
- ❌ Not logging operations for audit trail
- ❌ Proceeding with cleanup when decision state is `resume` or `abort`
