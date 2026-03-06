---
name: step-10-pre-merge-gate
description: "Check for git conflicts with base branch and present merge candidates"
nextStepFile: './step-11-human-approval-gate.md'
phase: sequential
phase_number: 3
executor: orchestrator
quality_gate: pre-merge
---

# Step 10: Pre-Merge Gate

**Progress: Step 10 of 14** — Next: Human Approval Gate
**Phase:** 3 (SEQUENTIAL — Orchestrator Only)
**Quality Gate:** Pre-Merge Validation

---

## STEP GOAL

Verify that each story branch can merge cleanly with the base branch. Detect conflicts BEFORE presenting branches to the user for approval. This is the final technical gate before human decision-making.

---

## MANDATORY EXECUTION RULES

- 🛑 Check EVERY approved branch for merge conflicts
- 📖 Use dry-run merge — do NOT actually merge anything
- 🚫 NEVER present conflicting branches as "ready" without flagging
- 🎯 Orchestrator executes directly — no sub-agents

---

## EXECUTION SEQUENCE

### 1. Check Base Branch for Changes

First, check if the base branch has moved since we started:

```bash
# Current base branch HEAD
git rev-parse {base_branch}

# Compare to the base commit from worktree creation (step 3)
# worktree_registry.base_commit
```

**IF base branch has new commits:**
- Warn user: "Base branch has {n} new commits since workflow started"
- This increases conflict probability
- Offer: rebase worktrees or proceed as-is

### 2. Test Merge for Each Branch

For EACH approved story branch, simulate a merge:

```bash
# Create a temporary merge test (does NOT modify the worktree or base branch)
git -C {worktree_path} merge-tree $(git merge-base {base_branch} {story-id}) {base_branch} {story-id}
```

**Alternative approach if merge-tree isn't available:**

```bash
# Dry-run merge in a temporary space
# 1. Note current position
git -C {worktree_path} stash list  # Verify clean state

# 2. Check if merge would conflict (without actually merging)
git -C {worktree_path} format-patch {base_branch}..HEAD --stdout | \
  git apply --check --directory={temp} 2>&1
```

**Simplest reliable approach:**

```bash
# Check for file-level conflicts by comparing changed files
# Story A's changed files:
git -C {worktree_path} diff --name-only {base_branch}..HEAD

# If base branch moved: check if any of those files changed on base too
git diff --name-only {base_commit}..{base_branch} 2>/dev/null
```

### 2.5 Pre-Merge Test Verification

Run the FULL test suite one final time in each approved worktree. This catches:
- Regressions introduced during corrective loop (step 7)
- Test failures from shared file reverts (step 9)
- Flaky tests that passed initially but fail on re-run

```bash
# For each approved worktree:
cd {worktree_path} && npm test 2>&1  # Or project-appropriate command
```

**Results classification:**
| Condition | Classification |
|-----------|---------------|
| Tests pass | ✅ MERGE_READY |
| Tests fail (were passing before) | ⚠️ REGRESSION_DETECTED |
| Tests fail (known issue, force-approved) | ⚠️ KNOWN_FAILURE |
| Tests cannot run | ℹ️ TESTS_NOT_VERIFIABLE |

**IF regression detected:**
- Mark branch as `MERGE_BLOCKED`
- Include failure details in pre-merge report
- User must explicitly override to include in merge candidates

**Add to merge_candidates:**
```yaml
pre_merge_tests:
  status: "passed"  # or "regression" or "known_failure" or "not_verifiable"
  test_output_summary: string
```

### 3. Classify Merge Readiness

For each story branch:

| Condition | Classification |
|-----------|---------------|
| No conflicts detected, base unchanged | ✅ **CLEAN** |
| No conflicts detected, base moved but different files | ✅ **LIKELY_CLEAN** |
| Potential conflict — same files changed on base and branch | ⚠️ **CONFLICT_RISK** |
| Known conflict — shared file mutations flagged in step 9 | 🔴 **CONFLICT_EXPECTED** |
| Force-approved story | ⚠️ **FORCE_APPROVED** |

### 4. Determine Recommended Merge Order

If multiple branches are ready, suggest an optimal merge order:

**Strategy:** Merge simplest (fewest changes) first, then progressively complex:

```yaml
recommended_merge_order:
  - story_id: "{story-with-least-changes}"
    reason: "Fewest file changes, lowest conflict risk"
    files_changed: 3
  - story_id: "{story-medium-changes}"
    reason: "Medium complexity"
    files_changed: 8
  - story_id: "{story-most-changes}"
    reason: "Most changes, highest potential for conflicts with earlier merges"
    files_changed: 15
```

### 5. Compile Merge Candidates

```yaml
merge_candidates:
  - story_id: "{story-id-1}"
    branch: "{story-id-1}"
    merge_status: "clean"
    files_changed: int
    commits: int
    review_status: "approved"  # or "force_approved"
    shared_mutations: list[string]  # From step 9
    cross_validation_notes: list[string]  # From step 8
    recommended_order: 1
  - story_id: "{story-id-2}"
    # ...
```

### 6. Present Pre-Merge Report

```
🚦 PRE-MERGE GATE REPORT
═══════════════════════════════════════

Base branch: {base_branch}
{if base_moved}
⚠️ Base branch has moved: {new_commit_count} new commits since workflow start
{/if}

MERGE CANDIDATES (recommended order):

  1. ✅ {story-id-1} — CLEAN
     {commit_count} commits, {files_count} files changed
     Review: Approved
     Shared mutations: None

  2. ⚠️ {story-id-2} — CONFLICT RISK
     {commit_count} commits, {files_count} files changed
     Review: Approved
     Shared mutations: package.json (accepted by user)
     Note: Same files modified on base branch

  3. ⚠️ {story-id-3} — FORCE APPROVED
     {commit_count} commits, {files_count} files changed
     Review: Force approved (unresolved minor issues)
     Shared mutations: None

SUMMARY:
  Clean: {count}
  Conflict risk: {count}
  Force approved: {count}

All branches will be presented for individual approval in the next step.
```

### 7. Store Pre-Merge State

```yaml
pre_merge_state:
  base_moved: boolean
  base_new_commits: int
  merge_candidates: {complete list from above}
  recommended_order: list[string]
  actions_pending_approval: list  # For output contract
```

### 8. Proceed to Human Approval

```
✅ Pre-merge gate complete.
   {candidate_count} branches ready for your approval.
   Loading Step 11: Human Approval Gate...
```

Load, read completely, then execute `{nextStepFile}`.

---

## QUALITY GATE: PRE-MERGE

| Validates | Failure Action |
|-----------|----------------|
| No git conflicts with base branch | Alert user with conflict details |
| Shared mutations flagged | Include in candidate report |
| Force-approved stories flagged | Include in candidate report |

---

## SUCCESS METRICS

- ✅ Every branch tested for merge compatibility
- ✅ Base branch movement detected and reported
- ✅ Conflicts clearly identified with specifics
- ✅ Optimal merge order recommended
- ✅ Complete candidate profiles compiled

## FAILURE MODES

- ❌ Not checking if base branch moved
- ❌ Actually performing a merge (this is dry-run only!)
- ❌ Marking conflicting branches as clean
- ❌ Not including shared mutation/cross-validation context in candidate profiles
