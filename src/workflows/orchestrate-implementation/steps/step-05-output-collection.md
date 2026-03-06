---
name: step-05-output-collection
description: "Gather and validate implementation results from all sub-agents"
nextStepFile: './step-06-reviewer-dispatch.md'
phase: sequential
phase_number: 3
executor: orchestrator
max_rerun_attempts: 2
---

# Step 5: Output Collection

**Progress: Step 5 of 14** — Next: Reviewer Dispatch
**Phase:** 3 (SEQUENTIAL — Orchestrator Collects)

---

## STEP GOAL

Gather implementation results from all sub-agents, validate that commits exist and tests were run, and classify each story's implementation status. This feeds the reviewer dispatch and corrective loop.

---

## MANDATORY EXECUTION RULES

- 🛑 VERIFY commits exist in each worktree — don't trust reports alone
- 📖 Parse every sub-agent's implementation_report
- 🚫 NEVER assume success — independently verify

---

## EXECUTION SEQUENCE

### 1. Parse Sub-Agent Reports

For each entry in `{dispatch_results}`:

**IF sub-agent returned a valid `implementation_report`:**
- Parse the structured report
- Extract: status, files_changed, commit_hash, tests_passed, lint_clean

**IF sub-agent output is unstructured (no valid report):**
- Mark as `report_missing`
- Attempt to extract any useful information from raw output
- Fall back to git-based verification (step 2)

### 2. Independent Git Verification

For EACH worktree, independently verify the sub-agent's claims:

```bash
# Check if there are new commits beyond base
git -C {worktree_path} log --oneline {base_branch}..HEAD

# Get the latest commit hash
git -C {worktree_path} log -1 --format="%H %s"

# List files changed from base
git -C {worktree_path} diff --name-only {base_branch}..HEAD

# Check for uncommitted changes (sub-agent forgot to commit)
git -C {worktree_path} status --porcelain
```

**Build verified results:**

```yaml
verified_result:
  story_id: "{story-id}"
  reported_status: "{from report}"
  verified_status: "{from git verification}"
  has_commits: boolean
  commit_count: int
  latest_commit: "{hash}"
  files_changed: list[string]
  uncommitted_changes: boolean  # sub-agent forgot to commit
  tests_reported_passing: boolean
  lint_reported_clean: boolean
  discrepancies: list[string]   # differences between report and git reality
```

### 2.5 Unauthorized Push Detection

For EACH worktree, verify that no sub-agent pushed to remote without authorization:

```bash
# Check if branch exists on remote (it should NOT at this point)
git ls-remote --heads origin {branch_name} 2>/dev/null
```

**IF the branch exists on remote:**
```
🚨 SECURITY VIOLATION DETECTED
═══════════════════════════════════════

Branch {branch_name} exists on remote origin.
This means a sub-agent pushed without authorization.

This is a CONTRACT VIOLATION — sub-agents must NEVER push.

Options:
[D] Delete remote branch (git push origin --delete {branch_name})
[K] Keep remote branch (acknowledge violation)
[X] Abort workflow — investigate security breach
```

Record in collection_state:
```yaml
security_violations:
  - story_id: "{story-id}"
    violation: "unauthorized_push"
    branch: "{branch_name}"
    user_decision: "{D/K/X}"
```

### 3. Handle Uncommitted Changes

If a sub-agent left uncommitted changes:

```bash
# Check what's uncommitted
git -C {worktree_path} status --porcelain
git -C {worktree_path} diff --stat
```

**Decision:** Report to user — do NOT auto-commit. The sub-agent's contract required committing. This is a quality signal.

### 3.5 Independent Test Verification

For EACH worktree classified as having commits (not failed/timeout), 
independently run the project's test suite:

```bash
# Auto-detect test runner and execute
if [ -f "{worktree_path}/package.json" ]; then
  cd {worktree_path} && npm test 2>&1
elif [ -f "{worktree_path}/pyproject.toml" ] || [ -f "{worktree_path}/setup.py" ]; then
  cd {worktree_path} && python -m pytest 2>&1
elif [ -f "{worktree_path}/Makefile" ]; then
  cd {worktree_path} && make test 2>&1
fi
```

**Store results:**
```yaml
test_verification:
  story_id: "{story-id}"
  tests_independently_verified: boolean
  test_exit_code: int
  test_output_summary: string  # Last 50 lines of test output
```

**Classification impact:**
- Sub-agent reported passing AND independently verified → trust status
- Sub-agent reported passing BUT tests fail independently → **NEEDS_CORRECTION** 
  (sub-agent report was inaccurate — flag this explicitly)
- Sub-agent reported failing AND independently confirmed → FAILED (consistent)
- Tests could not be run (no test runner detected) → note as `tests_not_verifiable`

**Update the classification table** in section 4 to include:
| Report says completed + git confirms + tests independently pass | **READY_FOR_REVIEW** |
| Report says completed + tests independently FAIL | **NEEDS_CORRECTION** (test discrepancy) |
| Tests not verifiable (no runner detected) | **READY_FOR_REVIEW** with warning |

### 4. Classify Implementation Results

For each story, determine the overall status:

| Condition | Classification |
|-----------|---------------|
| Report says completed + git confirms commits + tests pass | **READY_FOR_REVIEW** |
| Report says completed but tests fail or lint dirty | **NEEDS_CORRECTION** |
| Report says completed but no commits found | **IMPLEMENTATION_INCOMPLETE** |
| Report says failed with details | **FAILED** |
| Report says blocked | **BLOCKED** |
| No report / timeout | **UNKNOWN** |
| Has uncommitted changes | **UNCOMMITTED_WORK** |

### 5. Build Collection Summary

```yaml
collection_summary:
  total_stories: int
  ready_for_review: list[{story_id, commit_hash, files_changed}]
  needs_correction: list[{story_id, issues}]
  failed: list[{story_id, reason}]
  blocked: list[{story_id, blocker}]
  unknown: list[{story_id}]
  uncommitted: list[{story_id, uncommitted_files}]
```

### 6. Present Collection Report to User

```
📦 OUTPUT COLLECTION REPORT
═══════════════════════════════════════

Stories Implemented: {total}

  Ready for Review:
  ✅ {story-id-1} — {commit_count} commits, {files_count} files changed
  ✅ {story-id-2} — {commit_count} commits, {files_count} files changed

  Needs Correction:
  ⚠️ {story-id-3} — Tests failing ({test_details})

  Failed:
  ❌ {story-id-4} — {failure_reason}

  {if uncommitted}
  ⚠️ Uncommitted Work Detected:
  📝 {story-id-5} — {count} files with uncommitted changes
     (Sub-agent did not follow commit protocol)
  {/if}

Summary: {ready_count} ready for review,
         {correction_count} need correction,
         {failed_count} failed
```

### 7. User Decision on Problem Cases

If there are failed/blocked/unknown/uncommitted stories:

```
How would you like to handle problem cases?

[P] Proceed with ready stories only — review them now
[R] Re-run failed stories (max {max_rerun_attempts} re-runs remaining)
    NOTE: Circuit breaker — after {max_rerun_attempts} re-runs, this option 
    is removed. You must choose [P], [M], or [A].
[M] Manual intervention — I'll fix these myself
[A] Abort — stop the workflow
```

**Re-run tracking:**
```yaml
rerun_tracking:
  rerun_count: 0
  max_reruns: "{max_rerun_attempts}"
```

After each re-run: increment `rerun_count`. IF `rerun_count >= max_reruns`, remove [R] option on next presentation.

- **P**: Continue with ready_for_review stories only
- **R**: Re-dispatch implementors for failed stories (loops back to step 4 for those)
- **M**: User fixes manually, then orchestrator re-verifies
- **A**: Abort workflow (offer cleanup)

### 8. Store Collection State

```yaml
collection_state:
  collection_summary: {from above}
  stories_for_review: list[string]  # Final list moving to review
  stories_deferred: list[string]    # Set aside for now
  correction_needed: list[string]   # Will enter corrective loop
```

### 9. Proceed to Review

```
✅ Output collection complete.
   {review_count} stories ready for code review.
   Loading Step 6: Reviewer Dispatch...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ Every sub-agent's output parsed and validated
- ✅ Git state independently verified for each worktree
- ✅ Discrepancies between reports and git reality flagged
- ✅ Uncommitted changes detected and reported
- ✅ User informed and decided on problem cases

## FAILURE MODES

- ❌ Trusting sub-agent reports without git verification
- ❌ Auto-committing uncommitted changes (violates sub-agent contract)
- ❌ Not checking for uncommitted work in worktrees
- ❌ Proceeding with failed stories without user decision
