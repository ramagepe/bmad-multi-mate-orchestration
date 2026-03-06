---
name: step-12-push-approved-branches
description: "Push only user-approved branches to remote"
nextStepFile: './step-13-cleanup.md'
phase: sequential
phase_number: 3
executor: orchestrator
security: critical
---

# Step 12: Push Approved Branches

**Progress: Step 12 of 14** — Next: Cleanup
**Phase:** 3 (SEQUENTIAL — Orchestrator Only)
**Security Level:** CRITICAL — Only push explicitly approved branches

---

## STEP GOAL

Push ONLY the branches that the user explicitly approved in step 11. Execute pushes one at a time, verify each, and report results. This is the only step in the entire workflow that executes `git push`.

---

## MANDATORY EXECUTION RULES

> **Note:** IF this step was skipped (user cancelled in step 11), downstream steps should use default push_state: `branches_pushed=[]`, `push_status='cancelled_by_human'`.

- 🛑 **ONLY** push branches in `{branches_approved}` — NO exceptions
- 📖 Push ONE branch at a time — verify before pushing next
- 🚫 NEVER force push — always use standard `git push`
- 🎯 If ANY push fails, STOP and report — don't continue blindly
- ⚠️ Double-check approval state is "confirmed" before ANY push

---

## EXECUTION SEQUENCE

### 1. Pre-Push Safety Check

```bash
# Verify we have the approved branch list
# Verify final_confirmation == "confirmed"
# Verify no accidental branches in the list
```

**IF `{final_confirmation}` is NOT "confirmed":**
- STOP — do not push anything
- This should never happen but is a safety check

### 2. Push Each Branch — ONE AT A TIME

For EACH branch in `{branches_approved}` (in recommended merge order):

```bash
# Navigate to the worktree
# Push the branch to remote
git -C {worktree_path} push origin {branch_name}
```

**After each push, verify:**

```bash
# Verify the branch exists on remote
git ls-remote --heads origin {branch_name}

# Verify push was successful (check exit code)
```

**Record the result:**

```yaml
push_result:
  story_id: "{story-id}"
  branch: "{branch_name}"
  status: "success"  # or "failed"
  remote_ref: "origin/{branch_name}"
  error: null  # or error message
```

### 3. Handle Push Failures

**IF a push fails:**

```
❌ Push failed for branch: {branch_name}

Error: {error_message}

Common causes:
- Remote branch already exists (someone else pushed)
- Network connectivity issue
- Permission denied
- Remote rejected (hooks, protected branch rules)

Options:
[R] Retry this push
[S] Skip this branch and continue with others
[F] Force push (⚠️ DESTRUCTIVE — overwrites remote)
[X] Stop pushing — keep remaining branches local
```

- **R**: Retry the push
- **S**: Skip, mark as `push_failed`, continue
- **F**: Only if user explicitly requests AND confirms understanding:
  ```
  ⚠️ FORCE PUSH will overwrite any existing content on origin/{branch_name}.
  Are you ABSOLUTELY sure? Type "FORCE" to confirm:
  ```
- **X**: Stop all remaining pushes

### 4. Present Push Results

```
📤 PUSH RESULTS
═══════════════════════════════════════

  ✅ {story-id-1} → origin/{story-id-1} — pushed successfully
  ✅ {story-id-3} → origin/{story-id-3} — pushed successfully
  {if failures}
  ❌ {story-id-2} → FAILED: {error_reason}
  {/if}

Pushed: {success_count}/{total_count}

{if all_success}
🎉 All approved branches pushed to remote!
   You can now create Pull Requests for these branches.
{/if}

{if has_pr_suggestion}
💡 Tip: Create PRs with:
  {for each pushed branch}
  gh pr create --head {branch_name} --base {base_branch} --title "feat: {story_title}"
  {/for}
{/if}
```

### 5. Store Push State

```yaml
push_state:
  pushes_attempted: int
  pushes_succeeded: int
  pushes_failed: int
  push_results: list[{story_id, branch, status, remote_ref, error}]
  branches_pushed: list[string]  # For output contract
```

### 6. Proceed to Cleanup

```
✅ Push step complete.
   Loading Step 13: Cleanup...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ Only approved branches pushed — zero unauthorized pushes
- ✅ Each push verified individually
- ✅ Push failures handled gracefully
- ✅ User clearly informed of results
- ✅ PR creation hints provided

## FAILURE MODES

- ❌ **CRITICAL**: Pushing a branch not in the approved list
- ❌ **CRITICAL**: Force pushing without explicit user confirmation
- ❌ Continuing to push after a failure without user decision
- ❌ Not verifying push success
