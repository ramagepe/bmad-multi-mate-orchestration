---
name: step-01-scan-worktrees
description: "Scan all git worktrees under worktree_base_path, identify orphaned and active ones"
nextStepFile: './step-02-scan-branches.md'
phase: sequential
phase_number: 1
executor: orchestrator
---

# Step 1: Scan Worktrees

**Progress: Step 1 of 6** — Next: Scan Branches
**Phase:** Sequential — Orchestrator Only

---

## STEP GOAL

Discover all git worktrees under `{worktree_base_path}` and classify each as **active**, **orphaned**, or **stale**. This scan IS the state reconstruction — no separate state file is needed. The scan result becomes the source of truth for all downstream recovery decisions.

---

## MANDATORY EXECUTION RULES

- 🛑 NEVER modify or delete anything in this step — scan ONLY
- 📖 Read the output of every git command completely before classifying
- 🚫 NEVER assume a worktree is orphaned without checking — verify each one
- 🎯 This step runs SEQUENTIALLY — orchestrator only, no sub-agents

---

## EXECUTION SEQUENCE

### 1. Resolve and Validate Configuration

Load recovery configuration from bmo config:

```yaml
worktree_base_path: "{project-root}/.claude/worktrees"  # From bmo/config.yaml
```

Resolve `{project-root}` to the actual project root:

```bash
# Get the project root
git rev-parse --show-toplevel
```

Set `{worktree_base_path}` to `{project-root}/.claude/worktrees`.

**🛑 VALIDATE worktree_base_path (NON-NEGOTIABLE):**

```
1. worktree_base_path MUST be inside {project-root}
2. worktree_base_path MUST NOT be {project-root} itself
3. worktree_base_path MUST NOT be /, /home, /tmp, /usr, /etc, or any system directory
4. worktree_base_path SHOULD contain "worktree" or "claude" in the path (BMO convention)

IF any validation fails:
  → STOP workflow with error: "Invalid worktree_base_path: {value}. Aborting recovery."
  → Do NOT proceed to any scan or cleanup operations
```

**Verify we are in a git repository:**

```bash
git rev-parse --git-dir 2>/dev/null || echo "NOT_A_GIT_REPO"
```

**IF `NOT_A_GIT_REPO`:** STOP workflow — "Fatal: not a git repository. Cannot perform recovery."

### 2. Scan Git Worktree Registry

Run the canonical git worktree scan:

```bash
# List ALL worktrees known to git (includes main worktree)
git worktree list --porcelain
```

This returns blocks like:
```
worktree /path/to/main
HEAD abc1234
branch refs/heads/main

worktree /path/to/worktree-1
HEAD def5678
branch refs/heads/story-id-1
```

Parse EACH worktree entry to extract:
- `path` — absolute filesystem path
- `HEAD` — commit hash
- `branch` — branch reference (or `detached` if HEAD is detached)

### 3. Check Worktree Base Directory

Verify the worktree base directory exists:

```bash
# Check if worktree base exists
ls -la {worktree_base_path} 2>/dev/null || echo "DIRECTORY_NOT_FOUND"
```

**IF `DIRECTORY_NOT_FOUND`:**
- No BMO worktrees exist — record empty worktree scan with zero entries
- **Always proceed to step 2** (branch scan) — there may be orphan branches without worktrees
- The fast-path exit happens in step 3 (present status) if BOTH scans find nothing

**IF directory exists:**

```bash
# List all directories under worktree base
ls -1d {worktree_base_path}/*/ 2>/dev/null || echo "NO_SUBDIRECTORIES"
```

### 4. Cross-Reference Filesystem with Git Registry

For EACH directory found in `{worktree_base_path}`:

```bash
# Check if this directory is a valid git worktree
git -C {worktree_path} rev-parse --git-dir 2>/dev/null || echo "NOT_A_GIT_WORKTREE"

# Check the branch
git -C {worktree_path} branch --show-current 2>/dev/null || echo "DETACHED_OR_INVALID"

# Check for uncommitted changes
git -C {worktree_path} status --porcelain 2>/dev/null || echo "STATUS_FAILED"

# Check the last commit date
git -C {worktree_path} log -1 --format="%ci %s" 2>/dev/null || echo "NO_COMMITS"
```

### 5. Classify Each Worktree

Apply classification rules:

| Classification | Criteria |
|----------------|----------|
| **ACTIVE** | In git registry AND directory exists AND has recent commits (< 24h) |
| **ORPHANED** | Directory exists in `{worktree_base_path}` but NOT in git registry |
| **STALE** | In git registry AND directory exists BUT no commits in > 24h AND no uncommitted changes |
| **LOCKED** | Git reports the worktree as locked (`git worktree list` shows `locked`) |
| **PRUNABLE** | In git registry but directory does NOT exist (stale reference) |

**For each classified worktree, record:**

```yaml
worktree_entry:
  path: "{absolute_path}"
  branch: "{branch_name}"
  classification: "{active|orphaned|stale|locked|prunable}"
  head_commit: "{commit_hash}"
  last_commit_date: "{date}"
  has_uncommitted_changes: boolean
  uncommitted_file_count: int
  is_in_git_registry: boolean
  is_on_filesystem: boolean
```

### 6. Check for Prunable References

```bash
# Check for stale worktree references that git can auto-prune
# NOTE: `git worktree list --porcelain` does NOT emit "prunable" — use prune --dry-run
git worktree prune --dry-run 2>&1
```

Parse the output: each line starting with "Removing" indicates a prunable reference.
Count the lines to determine `prunable_count`. If no output, `prunable_count = 0`.

If prunable references exist, note them — they'll be cleaned in step 5.

### 7. Compile Scan Summary

Build the complete worktree scan result:

```yaml
worktree_scan:
  scan_timestamp: "{timestamp}"
  worktree_base_path: "{worktree_base_path}"
  main_worktree: "{main_worktree_path}"
  total_found: int
  classification_counts:
    active: int
    orphaned: int
    stale: int
    locked: int
    prunable: int
  worktrees: list[worktree_entry]  # All classified entries
  has_uncommitted_work: boolean     # True if ANY worktree has uncommitted changes
```

### 8. Store State

```yaml
recovery_state:
  worktree_scan: {complete scan from above}
  scan_phase: "worktrees_scanned"
```

### 9. Proceed to Next Step

```
✅ Worktree scan complete.
   Found {total_found} worktrees: {active} active, {orphaned} orphaned, {stale} stale, {prunable} prunable
   Loading Step 2: Scan Branches...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ Every directory under `{worktree_base_path}` was inspected
- ✅ Every git-registered worktree was cross-referenced with filesystem
- ✅ Each worktree classified with correct status
- ✅ Uncommitted changes detected and recorded (not modified)
- ✅ Complete scan state stored for downstream steps

## FAILURE MODES

- ❌ Modifying or deleting worktrees during scan phase
- ❌ Missing worktrees that exist on filesystem but not in git registry
- ❌ Not checking for uncommitted changes (would cause data loss in cleanup)
- ❌ Assuming empty directory means no worktrees (git registry may have stale refs)
