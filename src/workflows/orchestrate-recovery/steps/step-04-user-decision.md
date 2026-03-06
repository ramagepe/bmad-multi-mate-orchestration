---
name: step-04-user-decision
description: "User chooses recovery action: cleanup all, cleanup selective, resume, or abort"
nextStepFile: './step-05-execute-cleanup.md'
phase: sequential
phase_number: 1
executor: orchestrator
security: critical
---

# Step 4: User Decision

**Progress: Step 4 of 6** — Next: Execute Cleanup
**Phase:** Sequential — Orchestrator Only
**Security Level:** CRITICAL — User must explicitly choose action

---

## STEP GOAL

Present recovery options to the user and capture their decision. The user decides whether to clean up all artifacts, selectively clean specific items, resume a previous orchestration run, or abort recovery entirely. This is the primary decision gate — no cleanup happens without explicit user choice.

---

## MANDATORY EXECUTION RULES

- 🛑 NEVER auto-select a cleanup option — user MUST choose explicitly
- 📖 Present all options clearly with consequences explained
- 🚫 NEVER proceed to cleanup without a clear user decision
- 🎯 "Resume" means re-invoking `orchestrate-implementation` with existing worktree info — NOT magic state restoration
- ⚠️ Default is ABORT — no action taken unless user explicitly selects otherwise

---

## SECURITY RULES — NON-NEGOTIABLE

```
1. NEVER execute cleanup without explicit user selection
2. "Cleanup All" requires typing "DELETE" to confirm (destructive action)
3. Always explain what will be lost before any destructive option
4. Abort is always available — no pressure to clean up
5. Resume clearly states it is a RE-EXECUTION, not a continuation
```

---

## EXECUTION SEQUENCE

### 1. Load Recovery State

Retrieve `{recovery_state}` from step 3 including:
- Complete scan data (worktrees + branches)
- Risk assessment counts
- Presentation state

### 2. Present Decision Menu

```
🔧 BMO RECOVERY — CHOOSE ACTION
═══════════════════════════════════════════════════

Based on the scan results, here are your options:

[A] CLEANUP ALL — Remove ALL BMO worktrees and branches
    ⚠️ This removes {total_count} worktrees and {branch_count} branches
    {if uncommitted_work_detected}
    ⚠️ WARNING: {uncommitted_count} worktrees have UNCOMMITTED WORK that will be LOST
    {/if}
    {if unpushed_count > 0}
    ⚠️ WARNING: {unpushed_count} branches have NOT been pushed — commits will be LOST
    {/if}

[S] SELECTIVE CLEANUP — Choose which items to clean
    Review each worktree/branch individually
    Approve or skip each item

[R] RESUME — Re-invoke orchestration using existing worktrees
    This will pass existing worktree information to orchestrate-implementation
    for it to pick up where it left off. Not all worktrees may be resumable.
    {if uncommitted_work_detected}
    ℹ️ Worktrees with uncommitted changes will need manual attention first.
    {/if}

[X] ABORT — Do nothing, exit recovery
    All worktrees and branches remain as-is.
    You can run recovery again later.
```

### 3. Handle User Response

#### Option [A] — Cleanup All

**Extra safety gate — typing confirmation required:**

```
⚠️ CLEANUP ALL CONFIRMATION
═══════════════════════════════════════════════════

You are about to remove:
  🗑️ {worktree_count} worktrees
  🗑️ {branch_count} local branches
  {if prunable_count > 0}
  🗑️ {prunable_count} stale references
  {/if}

{if uncommitted_work_detected}
⚠️ PERMANENT DATA LOSS WARNING:
  The following worktrees have uncommitted work:
  {for each worktree with uncommitted work:}
    • {branch_name}: {uncommitted_file_count} uncommitted files
  {/for}
  This work will be PERMANENTLY LOST.
{/if}

{if unpushed_count > 0}
⚠️ UNPUSHED COMMITS WARNING:
  The following branches have not been pushed:
  {for each unpushed branch:}
    • {branch_name}: {commits_ahead} commits, {files_changed} files changed
  {/for}
  These commits will be PERMANENTLY LOST.
{/if}

Type "DELETE" to confirm cleanup of ALL items, or anything else to cancel:
```

**IF user types "DELETE":**
- Set `action = cleanup_all`
- Set `targets = all_worktrees + all_branches`
- Proceed to step 5

**IF user types anything else:**
- Return to decision menu

#### Option [S] — Selective Cleanup

Present each item individually:

```
🔧 SELECTIVE CLEANUP — Item {n} of {total}
═══════════════════════════════════════════════════

{for each worktree/branch pair:}
───────────────────────────────────────
Branch: {branch_name}
Worktree: {worktree_path}
Status: {classification}
Push status: {pushed|not_pushed}
Uncommitted changes: {yes — N files | no}
Commits ahead of base: {count}
Last commit: {date} — "{message}"
───────────────────────────────────────

[R] Remove — delete worktree and branch
[W] Remove worktree only — keep the branch
[K] Keep — do not touch
{/for}
```

After all items reviewed:

```
📋 SELECTIVE CLEANUP PLAN
═══════════════════════════════════════════════════

Will remove:
  🗑️ {list of items marked for removal}

Will keep:
  📌 {list of items marked to keep}

{if any_at_risk_items_marked_for_removal}
⚠️ Items with uncommitted work marked for removal:
  {list at-risk items}
  Type "CONFIRM" to proceed, or "BACK" to revise selections:
{else}
  Proceed with cleanup? [Y] Yes / [N] No / [B] Back to revise
{/if}
```

- Set `action = cleanup_selective`
- Set `targets = selected_items`
- Proceed to step 5

#### Option [R] — Resume

```
🔄 RESUME — Re-invoke Orchestration
═══════════════════════════════════════════════════

Resume will collect information about existing worktrees
and pass it to orchestrate-implementation for re-execution.

ℹ️ IMPORTANT: This is NOT magic state restoration.
   Resume means the orchestrator will:
   1. Use existing worktree paths and branch names
   2. Re-invoke orchestrate-implementation with this context
   3. The implementation workflow will pick up from where it can

{if uncommitted_work_detected}
⚠️ Worktrees with uncommitted changes may need manual attention:
{for each worktree with uncommitted work:}
  • {branch_name}: {uncommitted_file_count} uncommitted files
    You may need to commit or stash changes before resuming.
{/for}
{/if}

Resumable worktrees:
{for each active/stale worktree:}
  ✅ {branch_name} — {worktree_path}
     Commits: {count}, Changes: {clean|uncommitted}
{/for}

{if orphaned items exist}
Non-resumable (will need cleanup first):
{for each orphaned item:}
  ❌ {item} — {reason}
{/for}
{/if}

Proceed with resume? [Y] Yes / [N] No (return to menu)
```

**IF user confirms resume:**
- Set `action = resume`
- Compile worktree information for orchestrate-implementation:

```yaml
resume_context:
  existing_worktrees:
    - story_id: "{inferred_story_id}"
      branch: "{branch_name}"
      worktree_path: "{path}"
      has_commits: boolean
      has_uncommitted_work: boolean
  non_resumable: list[string]
```

- **Report the resume context to the user** and instruct them to invoke `orchestrate-implementation` (`{project-root}/_bmad/bmo/workflows/orchestrate-implementation/workflow.md`) with this information
- **END the recovery workflow** — recovery's job is done once resume info is collected

> **v1 LIMITATION:** The resume context is presented to the user for manual re-invocation of `orchestrate-implementation`. There is no automated handoff because implementation's input contract does not yet accept a `resume_context` field. The user provides the same epic + stories to OD, and step 3 (worktree setup) of implementation will detect existing worktrees and reuse them. A formal resume contract is planned for v2.

#### Option [X] — Abort

```
ℹ️ ABORT — No changes made.

All worktrees and branches remain as-is.
You can run [CW] Cleanup Worktrees again at any time.
```

- **END the recovery workflow**

### 4. Store Decision State

```yaml
recovery_state:
  # ... previous state preserved ...
  decision:
    action: "{cleanup_all|cleanup_selective|resume|abort}"
    targets:
      worktrees_to_remove: list[string]
      branches_to_delete: list[string]
      items_to_preserve: list[string]
    force: boolean  # True only if user confirmed deletion of at-risk items
    requires_force_delete: list[string]  # Branches that need -D instead of -d
  scan_phase: "decision_made"
```

### 5. Route to Next Step

- **IF `action == cleanup_all` or `action == cleanup_selective`:**
  ```
  ✅ Decision recorded. Loading Step 5: Execute Cleanup...
  ```
  Load, read completely, then execute `{nextStepFile}`.

- **IF `action == resume`:**
  Present resume context and END workflow.

- **IF `action == abort`:**
  END workflow — no further steps.

---

## SUCCESS METRICS

- ✅ All options clearly presented with consequences
- ✅ "Cleanup All" required typing "DELETE" to confirm
- ✅ Selective cleanup allowed per-item review
- ✅ Resume clearly explained as re-execution, not magic
- ✅ User explicitly chose an action — nothing auto-selected
- ✅ At-risk items required extra confirmation

## FAILURE MODES

- ❌ **CRITICAL**: Auto-selecting cleanup without user confirmation
- ❌ **CRITICAL**: Not requiring "DELETE" confirmation for cleanup-all with at-risk items
- ❌ Presenting resume as seamless continuation (it's a re-invocation)
- ❌ Not showing data loss warnings for uncommitted/unpushed work
- ❌ Not offering abort as an option
- ❌ Proceeding to cleanup step when user chose resume or abort
