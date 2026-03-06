---
name: step-09-shared-file-mutation-check
description: "Diff each worktree against snapshot to detect shared file mutations"
nextStepFile: './step-10-pre-merge-gate.md'
phase: sequential
phase_number: 3
executor: orchestrator
---

# Step 9: Shared File Mutation Check

**Progress: Step 9 of 14** — Next: Pre-Merge Gate
**Phase:** 3 (SEQUENTIAL — Orchestrator Only)

---

## STEP GOAL

Compare each worktree's state against the baseline snapshot captured in step 2. Detect any mutations to shared/critical files that were NOT part of the story's intended scope. Flag all mutations to the user — shared file changes are the #1 source of merge conflicts in parallel implementations.

---

## MANDATORY EXECUTION RULES

- 🛑 Compare against the snapshot from step 2 — NOT against current base branch
- 📖 Check EVERY file in the snapshot, in EVERY worktree
- 🚫 NEVER auto-resolve shared file conflicts — always flag to user
- 🎯 Orchestrator executes this directly — no sub-agents needed

---

## EXECUTION SEQUENCE

### 1. Load Snapshot from Step 2

Retrieve `{shared_file_snapshot}` captured in step 2:

```yaml
snapshot:
  base_commit: "{hash}"
  files:
    - path: "package.json"
      hash: "{original_hash}"
    - path: "src/types/index.ts"
      hash: "{original_hash}"
    # ...
```

### 2. Check Each Worktree Against Snapshot

For EACH approved worktree, compare shared files:

```bash
# For each file in the snapshot:
git -C {worktree_path} hash-object {file_path} 2>/dev/null || echo "DELETED"
```

**Compare hashes:**

```
IF worktree_hash == snapshot_hash:
  → File unchanged (expected for shared files) ✅

IF worktree_hash != snapshot_hash:
  → MUTATION DETECTED ⚠️
  → Record: which file, which story, what changed

IF worktree_hash == "DELETED" AND snapshot existed:
  → DELETION DETECTED 🔴
  → Record: file was deleted by this story

IF file exists in worktree but NOT in snapshot:
  → NEW FILE in shared location ℹ️
  → May be intentional — flag for review
```

### 3. Generate Detailed Diffs for Mutations

For each detected mutation, generate the actual diff:

```bash
# Show what changed in the shared file
git -C {worktree_path} diff {base_commit}..HEAD -- {shared_file_path}
```

### 4. Classify Mutations

```yaml
mutations:
  - file: "package.json"
    mutation_type: "modified"
    stories_affected: ["{story-id-1}", "{story-id-3}"]
    severity: "high"  # high: lock files, configs | medium: types, schemas | low: docs
    intentional: unknown  # Was this file in the story's scope?
    diff_summary: "Added dependency: @types/node"
    conflict_risk: "high"  # If multiple stories mutated the same file
```

**Severity classification:**

| File Type | Severity | Reason |
|-----------|----------|--------|
| Lock files (package-lock.json, etc.) | 🔴 HIGH | Almost guaranteed merge conflict |
| Package manifests (package.json, etc.) | 🔴 HIGH | Dependency conflicts likely |
| Config files (tsconfig, eslint, etc.) | 🟡 MEDIUM | May cause build issues |
| Shared types/schemas | 🟡 MEDIUM | Interface breaking changes |
| Documentation | 🟢 LOW | Usually safe to resolve |

### 5. Detect Multi-Story Conflicts

Check if the SAME shared file was mutated by MULTIPLE stories:

```
IF mutations.filter(file == X).stories.count > 1:
  → CONFLICT RISK: Multiple stories modified the same shared file
  → Generate comparison diff between the story versions
```

For each multi-story conflict:

```bash
# Compare what story A did vs what story B did to the same file
diff <(git -C {worktree_A} show HEAD:{file}) <(git -C {worktree_B} show HEAD:{file})
```

### 6. Present Mutation Report

```
📋 SHARED FILE MUTATION CHECK
═══════════════════════════════════════

Files checked: {snapshot_file_count}
Worktrees checked: {worktree_count}
Mutations found: {mutation_count}

{if no mutations}
✅ No shared file mutations detected. All clear!
{/if}

{if mutations exist}
⚠️ MUTATIONS DETECTED:

🔴 HIGH SEVERITY:
  • package.json — modified by {story-id-1}
    Change: Added dependency @types/node
    In story scope: {yes/no/unknown}

  • package-lock.json — modified by {story-id-1}, {story-id-3}
    ⚠️ CONFLICT RISK: 2 stories modified this file
    Merge will likely conflict — manual resolution needed

🟡 MEDIUM SEVERITY:
  • src/types/index.ts — modified by {story-id-2}
    Change: Added UserDTO type export
    In story scope: {yes/no/unknown}

{if multi_story_conflicts}
🔴 MULTI-STORY CONFLICTS:
  • package-lock.json: {story-id-1} and {story-id-3} both modified
    Resolution: Merge one first, then regenerate lock for the other
{/if}
{/if}
```

### 7. User Decision on Mutations

```
How would you like to handle shared file mutations?

[A] Accept all — I'll resolve conflicts during merge
[R] Review each — let me decide per mutation
[V] Revert unintended — revert mutations not in story scope
[X] Abort
```

- **A**: Flag all as acknowledged, proceed
- **R**: Present each mutation for individual accept/revert decision
- **V**: For each mutation NOT in the story's scope, revert it:
  ```bash
  git -C {worktree_path} checkout {base_commit} -- {file_path}
  git -C {worktree_path} add {file_path}
  git -C {worktree_path} commit -m "revert: unintended shared file mutation ({file_path})"
  ```
  
  **⚠️ IMPORTANT:** Reverting shared file mutations changes the worktree state AFTER 
  the reviewer approved it. The reviewer approved a different state than what now exists.
  
  Stories with reverted mutations are flagged as `needs_re_review: true` in mutation_state.
  The pre-merge gate (step 10) will note this, and the human approval gate (step 11) 
  will display a warning that these stories had post-review modifications.

- **X**: Abort

### 8. Store Mutation State

```yaml
mutation_state:
  mutations_detected: list[{file, stories, severity, decision}]
  multi_story_conflicts: list[{file, stories, resolution_needed}]
  mutations_reverted: list[{file, story, revert_commit}]
  mutations_accepted: list[{file, story}]
  shared_file_mutations: list  # For output contract
  stories_needing_re_review: list[{story_id, reverted_files}]
```

### 9. Proceed to Pre-Merge Gate

```
✅ Shared file mutation check complete.
   {mutation_count} mutations {handled/accepted}.
   Loading Step 10: Pre-Merge Gate...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ Every snapshot file checked in every worktree
- ✅ Mutations detected with accurate diffs
- ✅ Multi-story conflicts explicitly identified
- ✅ User informed and decided on each mutation category
- ✅ Reverts committed cleanly when requested

## FAILURE MODES

- ❌ Comparing against current base branch instead of snapshot
- ❌ Missing multi-story conflicts on the same file
- ❌ Auto-resolving shared file mutations without user input
- ❌ Not generating diffs for detected mutations
