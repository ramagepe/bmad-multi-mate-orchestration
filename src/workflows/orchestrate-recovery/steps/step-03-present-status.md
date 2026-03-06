---
name: step-03-present-status
description: "Present comprehensive recovery status to user — what was found, what's at risk"
nextStepFile: './step-04-user-decision.md'
phase: sequential
phase_number: 1
executor: orchestrator
---

# Step 3: Present Status

**Progress: Step 3 of 6** — Next: User Decision
**Phase:** Sequential — Orchestrator Only

---

## STEP GOAL

Present a clear, comprehensive view of the recovery landscape to the user. Show all discovered worktrees, branches, their statuses, and any risks. The user must understand exactly what exists before making cleanup/resume decisions. This step is READ-ONLY — display information only, no modifications.

---

## MANDATORY EXECUTION RULES

- 🛑 NEVER modify anything — this step is purely informational
- 📖 Present ALL findings from steps 1 and 2 — hide nothing
- 🚫 NEVER make recommendations that downplay data loss risks
- 🎯 Highlight uncommitted work prominently — it's the primary risk factor

---

## EXECUTION SEQUENCE

### 1. Load Complete Scan State

Retrieve `{recovery_state}` from step 2 containing:
- `worktree_scan` — all worktree findings
- `branch_scan` — all branch findings
- `correlation` — matched pairs and orphans
- `uncommitted_work_detected` — global flag

### 2. Handle Empty State (Fast Path)

**IF no BMO artifacts found** (`correlation.no_bmo_artifacts == true`):

```
🔍 RECOVERY SCAN — No BMO Artifacts Found
═══════════════════════════════════════

No BMO worktrees or branches were detected.
The environment is clean — no recovery needed.

Worktree base path: {worktree_base_path}
Main worktree: {main_worktree_path}

Nothing to clean up. ✅
```

**Produce the output contract with empty/default values before ending:**

```yaml
output:
  worktrees_found: []
  worktrees_cleaned: []
  branches_deleted: []
  uncommitted_work_detected: false
  recovery_status: "clean"
```

**Store state and END workflow** — no need to proceed to steps 4-6.

### 3. Present Overview Dashboard

```
🔍 BMO RECOVERY — STATUS REPORT
═══════════════════════════════════════════════════

Scan completed: {timestamp}
Worktree base: {worktree_base_path}

┌─────────────────────────────────────────────────┐
│  SUMMARY                                         │
├─────────────────────────────────────────────────┤
│  Worktrees found:        {total_worktrees}       │
│  BMO branches found:     {total_branches}        │
│  Pushed to remote:       {pushed_count}          │
│  Unpushed (local only):  {unpushed_count}        │
│  With uncommitted work:  {uncommitted_count}     │
└─────────────────────────────────────────────────┘
```

### 4. Present Detailed Worktree Table

**IF matched pairs exist** (branch + worktree):

```
🌳 WORKTREES WITH BRANCHES
═══════════════════════════════════════════════════

┌──────────────────┬────────────┬───────────┬──────────┬───────────────────┐
│ Branch           │ Worktree   │ Push      │ Changes  │ Classification    │
│                  │ Status     │ Status    │          │                   │
├──────────────────┼────────────┼───────────┼──────────┼───────────────────┤
│ {branch-1}       │ active     │ pushed    │ clean    │ ✅ Safe to remove │
│ {branch-2}       │ stale      │ not_pushed│ 3 files  │ ⚠️ Work at risk  │
│ {branch-3}       │ orphaned   │ pushed    │ clean    │ ✅ Safe to remove │
└──────────────────┴────────────┴───────────┴──────────┴───────────────────┘
```

### 5. Highlight At-Risk Work

**IF uncommitted work detected** (`uncommitted_work_detected == true`):

```
⚠️ UNCOMMITTED WORK DETECTED
═══════════════════════════════════════════════════

The following worktrees contain work that has NOT been committed or pushed.
Removing these worktrees will PERMANENTLY LOSE this work.

{for each worktree with uncommitted work:}
───────────────────────────────────────
⚠️ {branch_name} — {worktree_path}
───────────────────────────────────────
  Uncommitted changes: {uncommitted_file_count} files
  Type: {staged_changes|unstaged_changes|untracked_files|mixed}
  Push status: {push_status}
  Last commit: {last_commit_date} — "{last_commit_message}"
  Commits ahead of base: {commits_ahead}

  Changed files:
    {list of changed/untracked files}

{/for}
```

### 6. Show Orphaned Resources

**IF orphan worktrees exist** (worktree without branch):

```
👻 ORPHANED WORKTREES (no associated branch)
═══════════════════════════════════════════════════

These directories exist in {worktree_base_path} but have no
corresponding branch in git. They may be remnants of a failed cleanup.

{for each orphan_worktree:}
  📁 {worktree_path}
     Status: {classification}
     Has changes: {yes/no}
{/for}
```

**IF orphan branches exist** (branch without worktree):

```
🔀 ORPHANED BRANCHES (no associated worktree)
═══════════════════════════════════════════════════

These branches exist locally but have no worktree.
They may be from a previous run where worktrees were removed
but branches were kept.

{for each orphan_branch:}
  🔀 {branch_name}
     Push status: {pushed|not_pushed}
     Merge status: {merged|unmerged}
     Commits ahead: {count}
     Last commit: {date} — "{message}"
{/for}
```

### 7. Show Prunable References

**IF prunable worktree references exist:**

```
🗑️ STALE GIT REFERENCES
═══════════════════════════════════════════════════

Git has worktree references pointing to non-existent directories.
These can be safely pruned with `git worktree prune`.

  Count: {prunable_count}
```

### 8. Present Risk Assessment

```
📊 RISK ASSESSMENT
═══════════════════════════════════════════════════

Safe to clean (pushed, no uncommitted work):    {safe_count}
At risk (uncommitted work or unpushed):          {at_risk_count}
Prunable (stale references only):                {prunable_count}

{if at_risk_count > 0}
⚠️ {at_risk_count} items require careful handling.
   You will be asked to confirm before any at-risk work is removed.
{else}
✅ All items are safe to clean up.
{/if}
```

### 9. Store Presentation State

```yaml
recovery_state:
  # ... previous state preserved ...
  presentation:
    safe_to_clean_count: int
    at_risk_count: int
    prunable_count: int
    empty_scan: boolean
  scan_phase: "status_presented"
```

### 10. Proceed to Next Step

```
Status presentation complete.
Loading Step 4: User Decision...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ All scan findings presented clearly and completely
- ✅ Uncommitted work highlighted prominently with file details
- ✅ Risk assessment clearly distinguishes safe vs at-risk items
- ✅ User has complete picture before making decisions
- ✅ Empty state handled gracefully with fast-path exit

## FAILURE MODES

- ❌ Hiding or minimizing uncommitted work warnings
- ❌ Not showing push status (user can't assess data loss risk)
- ❌ Modifying or cleaning anything during presentation
- ❌ Not handling the empty-scan fast path
- ❌ Presenting information in a confusing or incomplete way
