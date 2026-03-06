---
name: step-11-human-approval-gate
description: "User approves/rejects push for each branch — NEVER auto-push"
nextStepFile: './step-12-push-approved-branches.md'
phase: sequential
phase_number: 3
executor: orchestrator
security: critical
skipToStepFile: './step-13-cleanup.md'
skipCondition: "user cancelled or no branches approved"
---

# Step 11: Human Approval Gate

**Progress: Step 11 of 14** — Next: Push Approved Branches
**Phase:** 3 (SEQUENTIAL — Human Decision Point)
**Security Level:** CRITICAL — NO AUTOMATIC PUSHES EVER

---

## STEP GOAL

Present each merge candidate to the user for individual approval. The user decides which branches get pushed. This is the most critical security gate in the entire workflow — the orchestrator NEVER pushes without explicit, per-branch human confirmation.

---

## MANDATORY EXECUTION RULES

- 🛑 **ABSOLUTE RULE**: NEVER auto-push ANY branch under ANY circumstance
- 📖 Present EACH branch individually for approval
- 🚫 NEVER batch-approve without explicit user consent for batch mode
- 🎯 User must explicitly confirm EACH branch they want pushed
- ⚠️ Default action is SKIP — not push

---

## SECURITY RULES — NON-NEGOTIABLE

```
1. Push EXCLUSIVELY with explicit user confirmation
2. Each branch requires its own approval
3. "Approve all" requires explicit user request — never suggested by orchestrator
4. Rejection is always an option — no pressure to approve
5. User can review diffs before approving
6. No timeouts on approval — wait indefinitely
```

---

## EXECUTION SEQUENCE

### 1. Present Approval Overview

```
🔐 HUMAN APPROVAL GATE
═══════════════════════════════════════

{candidate_count} branches ready for your review and approval.
Each branch requires your explicit approval before pushing.

⚠️ DEFAULT: Branches will NOT be pushed unless you approve them.
   Unapproved branches will remain as local branches.

Recommended merge order:
{list from step 10}

How would you like to review?

[O] One-by-one — review each branch individually (recommended)
[S] Summary — see all at once, then decide
[D] Diff view — see full diffs before deciding
```

### 2. Individual Branch Approval

For EACH merge candidate (in recommended order):

```
───────────────────────────────────────
📋 Branch: {story-id}
───────────────────────────────────────

Story: {story title from story file}
Branch: {branch_name}
Merge Status: {clean / conflict_risk / force_approved}
Review Status: {approved / force_approved}
Commits: {count}
Files Changed: {count}
  {list of changed files}

{if shared_mutations}
⚠️ Shared File Mutations:
  {list mutations}
{/if}

{if cross_validation_notes}
ℹ️ Cross-Validation Notes:
  {list notes}
{/if}

{if force_approved}
⚠️ This story was FORCE APPROVED with known issues:
  {list remaining issues}
{/if}

───────────────────────────────────────

[Y] Approve push for this branch
[N] Skip — do NOT push this branch
[D] Show full diff before deciding
[R] Show review report before deciding
[?] Ask questions about this branch
```

### 3. Handle Each Response

- **Y**: Mark branch as `approved_for_push`
  ```
  ✅ {story-id} — APPROVED for push
  ```

- **N**: Mark branch as `skipped`
  ```
  ⏭️ {story-id} — Skipped (branch preserved locally)
  ```

- **D**: Show the full diff:
  ```bash
  git -C {worktree_path} diff {base_branch}..HEAD
  ```
  Then re-present the approval menu for this branch.

- **R**: Show the complete review report from step 6.
  Then re-present the approval menu for this branch.

- **?**: Answer user's questions about the branch.
  Then re-present the approval menu for this branch.

### 4. Confirmation Summary

After all branches have been reviewed:

```
🔐 APPROVAL SUMMARY
═══════════════════════════════════════

Approved for push:
  ✅ {story-id-1}
  ✅ {story-id-3}

Skipped (not pushing):
  ⏭️ {story-id-2} — will remain as local branch

───────────────────────────────────────

⚠️ FINAL CONFIRMATION: Push {approved_count} branches to remote?

This will execute:
  git push origin {story-id-1}
  git push origin {story-id-3}

[CONFIRM] Yes, push approved branches
[CANCEL]  Cancel — don't push anything
[EDIT]    Change my approvals
```

### 5. Handle Final Confirmation

- **CONFIRM**: Proceed to push step
- **CANCEL**: Skip pushing entirely — all branches remain local
- **EDIT**: Return to individual approval flow

### 6. Store Approval State

```yaml
approval_state:
  branches_approved: list[{story_id, branch}]
  branches_skipped: list[{story_id, branch, reason}]
  final_confirmation: "confirmed"  # or "cancelled"
  approval_timestamp: "{timestamp}"
```

### 7. Route to Next Step

- **IF confirmed and branches approved** → Load `{nextStepFile}` (step-12 — Push Approved Branches)
- **IF cancelled or no branches approved** → Load `{skipToStepFile}` (step-13 — Cleanup)

**IF confirmed and branches approved:**
```
✅ Approval confirmed. Loading Step 12: Push Approved Branches...
```
Load `{nextStepFile}`.

**IF cancelled or no branches approved:**
```
ℹ️ No branches will be pushed. Skipping to cleanup.
   Loading Step 13: Cleanup...
```
Load `{skipToStepFile}`.

Load, read completely, then execute the appropriate step file.

---

## SUCCESS METRICS

- ✅ Every branch presented individually for approval
- ✅ User had option to view diff and review report
- ✅ No branch pushed without explicit approval
- ✅ Final confirmation required before any push
- ✅ Skipped branches clearly documented

## FAILURE MODES

- ❌ **CRITICAL**: Auto-pushing any branch without user approval
- ❌ **CRITICAL**: Batch-approving without explicit user request
- ❌ Not presenting shared mutation/force-approval warnings
- ❌ Not offering diff/review viewing before approval
- ❌ Rushing user through approvals
