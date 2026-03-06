---
name: step-03-worktree-setup
description: "Create git worktrees and branches for each story — SEQUENTIAL, orchestrator only"
nextStepFile: './step-04-implementor-fanout.md'
phase: sequential
phase_number: 1
executor: orchestrator
security: critical
---

# Step 3: Worktree Setup (Sequential)

**Progress: Step 3 of 14** — Next: Implementor Fan-Out
**Phase:** 1 (SEQUENTIAL — Orchestrator Only)
**Security Level:** CRITICAL — Branch creation must complete before any sub-agent starts

---

## STEP GOAL

Create isolated git worktrees and branches for EACH story that passed readiness check. This MUST complete entirely before ANY sub-agent is dispatched. This is the core of the git race condition prevention strategy.

---

## MANDATORY EXECUTION RULES

- 🛑 **SECURITY CRITICAL**: ALL worktrees MUST be created BEFORE step 4 begins
- 📖 Create worktrees SEQUENTIALLY — one at a time, verify each
- 🚫 NEVER allow sub-agents to create branches — orchestrator ONLY
- 🎯 Use the golden pattern from the spike: `git worktree add`
- ⚠️ If ANY worktree creation fails, STOP and resolve before continuing

---

## CONFIGURATION

```yaml
worktree_base_path: "{worktree_base_path}"   # From bmo config
base_branch: "{base_branch}"                  # From input contract
stories: "{stories_proceeding}"               # From step 1 readiness state
```

---

## EXECUTION SEQUENCE

### 1. Pre-Flight Checks

Before creating any worktrees, verify the environment:

```bash
# Verify we're in the project root
git rev-parse --show-toplevel

# Verify base branch exists and is clean
git status --porcelain
# IF output is non-empty: warn user about uncommitted changes

# Verify base branch is up to date (informational only)
git log --oneline -1 {base_branch}

# Check for any existing worktrees that might conflict
git worktree list
```

**IF existing worktrees conflict with planned story branches:**
- List conflicts to user
- Offer options: clean up existing, use different branch names, or abort
- Reference orchestrate-recovery patterns for cleanup

### 2. Create Worktree Base Directory

```bash
# Ensure the base directory exists
mkdir -p {worktree_base_path}
```

### 3. Create Worktrees — SEQUENTIAL, ONE AT A TIME

**GOLDEN PATTERN** (validated in spike):

For EACH story in `{stories_proceeding}`, execute IN ORDER:

```bash
# Extract story-id from the story file path
# Convention: story file name or explicit ID becomes branch name

git worktree add {worktree_base_path}/{story-id} -b {story-id} {base_branch}
```

**After EACH worktree creation, verify:**

```bash
# Verify worktree exists
ls -la {worktree_base_path}/{story-id}

# Verify branch was created
git branch --list {story-id}

# Verify worktree is on correct branch
git -C {worktree_base_path}/{story-id} branch --show-current
```

**IF creation fails for a story:**
- Log the error with full details
- Attempt cleanup: `git worktree remove {worktree_base_path}/{story-id} 2>/dev/null`
- Add story to `{stories_failed}` list with reason
- Continue with remaining stories (don't abort entire workflow)

### 4. Verify All Worktrees

After ALL worktrees are created, run a final verification:

```bash
# List all worktrees — should show main + one per story
git worktree list
```

**Build the worktree registry:**

```yaml
worktree_registry:
  base_commit: "{git rev-parse HEAD}"
  created_at: "{timestamp}"
  worktrees:
    - story_id: "{story-id-1}"
      branch: "{story-id-1}"
      path: "{worktree_base_path}/{story-id-1}"
      status: created
      story_file: "{story_file_path_1}"
    - story_id: "{story-id-2}"
      branch: "{story-id-2}"
      path: "{worktree_base_path}/{story-id-2}"
      status: created
      story_file: "{story_file_path_2}"
```

### 5. Present Setup Summary to User

```
🌳 WORKTREE SETUP COMPLETE
═══════════════════════════════════════

Base: {base_branch} @ {short_commit_hash}
Worktree root: {worktree_base_path}

Created {success_count} of {total_count} worktrees:

  ✅ {story-id-1} → {worktree_base_path}/{story-id-1}
  ✅ {story-id-2} → {worktree_base_path}/{story-id-2}
  ✅ {story-id-3} → {worktree_base_path}/{story-id-3}
  {if any failed}
  ❌ {story-id-4} → FAILED: {reason}
  {/if}

All branches created from {base_branch}.
Sub-agents will ONLY edit files and commit — no branch operations.

Ready to dispatch implementors? [C] Continue / [X] Abort
```

### 6. User Confirmation

- **C**: Proceed to implementor fan-out
- **X**: Offer cleanup (remove all worktrees) and abort

**IF C selected and there were failures:**
- Confirm user wants to proceed with partial set
- Update `{stories_proceeding}` to exclude failed stories

### 7. Store Worktree State

Persist the complete worktree registry for all downstream steps:

```yaml
worktree_state:
  worktree_registry: {complete registry from above}
  stories_proceeding: list[string]  # Updated if any failed
  stories_failed_setup: list[{id, reason}]
```

### 8. Proceed to Fan-Out

```
✅ {count} worktrees ready for implementation.
   Loading Step 4: Implementor Fan-Out (PARALLEL)...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SECURITY RULES ENFORCED

1. ✅ Commits ONLY in assigned worktrees — branches are isolated
2. ✅ All worktrees/branches created BEFORE fan-out
3. ✅ Sub-agents will never create branches (enforced via contract in step 4)
4. ✅ Each worktree starts from same base commit

---

## RECOVERY PATTERNS

If worktree creation fails mid-way, use these commands:

```bash
# List current worktrees
git worktree list

# Remove a specific worktree
git worktree remove {worktree_base_path}/{story-id} --force

# Delete the branch (safe — only deletes if fully merged)
git branch -d {story-id}

# Force delete branch (if needed)
git branch -D {story-id}

# Prune stale worktree references
git worktree prune
```

---

## SUCCESS METRICS

- ✅ All worktrees created from same base commit
- ✅ Each worktree verified on correct branch
- ✅ No pre-existing branch conflicts
- ✅ Worktree registry complete and persisted
- ✅ User confirmed before proceeding to fan-out

## FAILURE MODES

- ❌ Creating worktrees in parallel (race condition risk)
- ❌ Not verifying each worktree after creation
- ❌ Proceeding to fan-out with failed worktrees in the list
- ❌ Not recording the base commit (needed for conflict detection later)
