---
name: step-02-scan-branches
description: "Scan BMO-created branches, detect uncommitted work, correlate with worktree scan"
nextStepFile: './step-03-present-status.md'
phase: sequential
phase_number: 1
executor: orchestrator
---

# Step 2: Scan Branches

**Progress: Step 2 of 6** — Next: Present Status
**Phase:** Sequential — Orchestrator Only

---

## STEP GOAL

Identify all branches created by BMO orchestration runs, check each for uncommitted work and push status, and correlate branch data with the worktree scan from step 1. This completes the state reconstruction — after this step, we know everything about the recovery landscape.

---

## MANDATORY EXECUTION RULES

- 🛑 NEVER delete or modify any branches — scan ONLY
- 📖 Check BOTH local and remote status for every BMO branch
- 🚫 NEVER assume a branch is safe to delete without checking push status
- 🎯 Correlate every branch with its worktree (if any) from step 1 data

---

## EXECUTION SEQUENCE

### 1. Load Worktree Scan State

Retrieve `{recovery_state.worktree_scan}` from step 1. This provides:
- Known worktree paths and their branches
- Which worktrees have uncommitted changes
- Classification of each worktree

### 2. Check Remote Accessibility

```bash
# Verify the remote is reachable before checking push status
git ls-remote --exit-code origin HEAD 2>/dev/null
```

**IF remote is NOT accessible:**
- Set `remote_accessible: false`
- All push status checks will return `"unknown"` instead of `"not_pushed"`
- Display warning: "⚠️ Remote 'origin' is not accessible. Push status will be reported as 'unknown'. Branches with unknown push status require extra caution before deletion."
- Branches with `push_status: "unknown"` should be treated as AT-RISK (same as unpushed)

**IF remote IS accessible:**
- Set `remote_accessible: true`
- Proceed with normal push status checks

### 3. Identify BMO Branch Naming Convention

BMO creates branches using story IDs. Scan for branches that match BMO patterns:

```bash
# List ALL local branches
git branch --list --format='%(refname:short) %(objectname:short) %(committerdate:short) %(upstream:track)'

# List branches that appear to be BMO-created
# BMO convention: branches named after story IDs (typically kebab-case with story prefix)
git branch --list --format='%(refname:short)'
```

**BMO branch identification heuristics:**
1. Branch has a corresponding worktree in `{worktree_base_path}` — **most reliable indicator**
2. Branch name matches story file naming patterns (e.g., `story-*`, task-based names)
3. Branch was created from the same base commit as other BMO branches

> **v1 LIMITATION:** Without a formal BMO branch naming prefix (e.g., `bmo/`), heuristic #1 is the only reliable indicator. Orphan branches (no worktree) may not be detected as BMO-created. This is an accepted limitation — the user can always selectively clean branches via standard git commands. A formal prefix convention is planned for v2.

### 4. Analyze Each Candidate Branch

For EACH branch identified as potentially BMO-created:

```bash
# Check if branch exists on remote
git branch -r --list "origin/{branch_name}"

# Check merge status relative to base branch
git log --oneline {base_branch}..{branch_name} 2>/dev/null | head -20

# Get commit count ahead of base
git rev-list --count {base_branch}..{branch_name} 2>/dev/null || echo "0"

# Get files changed relative to base
git diff --stat {base_branch}..{branch_name} 2>/dev/null

# Check if branch has been merged into base
git branch --merged {base_branch} | grep -w "{branch_name}" && echo "MERGED" || echo "UNMERGED"
```

### 5. Check for Uncommitted Work in Associated Worktrees

For each branch that has a corresponding worktree:

```bash
# Check for staged but uncommitted changes
git -C {worktree_path} diff --cached --stat 2>/dev/null

# Check for unstaged changes
git -C {worktree_path} diff --stat 2>/dev/null

# Check for untracked files
git -C {worktree_path} ls-files --others --exclude-standard 2>/dev/null
```

**Classify uncommitted work:**

| Status | Meaning | Risk Level |
|--------|---------|------------|
| **clean** | No uncommitted changes | Safe to remove |
| **staged_changes** | Changes staged but not committed | ⚠️ Work at risk |
| **unstaged_changes** | Modified files not staged | ⚠️ Work at risk |
| **untracked_files** | New files not added to git | ⚠️ Work at risk |
| **mixed** | Combination of above | ⚠️ Work at risk |

### 6. Determine Push Status for Each Branch

```bash
# Check if branch has remote tracking
git config --get "branch.{branch_name}.remote" 2>/dev/null || echo "NO_REMOTE"

# If remote exists, check if local is ahead
git rev-list --count "origin/{branch_name}..{branch_name}" 2>/dev/null || echo "NOT_PUSHED"
```

**Push status classification:**

| Status | Meaning |
|--------|---------|
| **pushed** | Branch exists on remote and local matches remote |
| **ahead** | Branch pushed but local has additional commits |
| **not_pushed** | Branch only exists locally |
| **no_remote** | No remote tracking configured |
| **unknown** | Remote not accessible — cannot determine push status (treat as AT-RISK) |

> **NOTE:** If `remote_accessible == false` (from section 2), ALL push statuses default to `"unknown"`. Branches with `unknown` push status require extra caution — they MUST be treated as unpushed for safety purposes.

### 7. Build Branch Registry

For EACH identified branch, compile:

```yaml
branch_entry:
  name: "{branch_name}"
  is_bmo_created: boolean
  has_worktree: boolean
  worktree_path: "{path or null}"
  commits_ahead_of_base: int
  files_changed: int
  push_status: "{pushed|ahead|not_pushed|no_remote}"
  merge_status: "{merged|unmerged}"
  uncommitted_work: "{clean|staged_changes|unstaged_changes|untracked_files|mixed}"
  uncommitted_file_count: int
  last_commit_date: "{date}"
  last_commit_message: "{message}"
```

### 8. Correlate Branches with Worktrees

Build a unified view matching branches to their worktrees:

```yaml
correlation:
  matched_pairs:     # Branch has worktree
    - branch: "{name}"
      worktree: "{path}"
      worktree_status: "{active|orphaned|stale}"
      branch_push_status: "{pushed|not_pushed}"
      has_uncommitted_work: boolean
  orphan_worktrees:  # Worktree exists but no branch (detached HEAD?)
    - worktree: "{path}"
      status: "{classification}"
  orphan_branches:   # Branch exists but no worktree
    - branch: "{name}"
      push_status: "{status}"
      merge_status: "{status}"
  no_bmo_artifacts:  # True if nothing found
    boolean
```

### 9. Store Complete Recovery State

```yaml
recovery_state:
  worktree_scan: {from step 1}
  branch_scan:
    scan_timestamp: "{timestamp}"
    total_branches_found: int
    bmo_branches: int
    pushed_branches: int
    unpushed_branches: int
    branches_with_uncommitted_work: int
    branches: list[branch_entry]
  correlation: {from above}
  uncommitted_work_detected: boolean  # TRUE if ANY branch/worktree has uncommitted work
  scan_phase: "fully_scanned"
```

### 10. Proceed to Next Step

```
✅ Branch scan complete.
   Found {bmo_branches} BMO branches: {pushed} pushed, {unpushed} unpushed
   Uncommitted work detected: {yes/no}
   Loading Step 3: Present Status...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ All local branches scanned and BMO-created ones identified
- ✅ Push status determined for every BMO branch
- ✅ Uncommitted work detected in all associated worktrees
- ✅ Branches correlated with worktrees from step 1
- ✅ Complete recovery state stored for downstream steps

## FAILURE MODES

- ❌ Missing branches that exist locally but were not identified as BMO-created
- ❌ Not checking remote push status (would cause data loss if branch deleted)
- ❌ Not detecting uncommitted changes (staged, unstaged, untracked)
- ❌ Modifying or deleting any branches during scan
- ❌ Not correlating branches with worktrees (incomplete picture for user)
