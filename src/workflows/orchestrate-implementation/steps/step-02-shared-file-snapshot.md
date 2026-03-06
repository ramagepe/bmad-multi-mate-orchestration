---
name: step-02-shared-file-snapshot
description: "Snapshot shared files before fan-out to detect post-implementation mutations"
nextStepFile: './step-03-worktree-setup.md'
phase: sequential
phase_number: 1
executor: orchestrator
---

# Step 2: Shared File Snapshot

**Progress: Step 2 of 14** — Next: Worktree Setup
**Phase:** 1 (SEQUENTIAL — Orchestrator Only)

---

## STEP GOAL

Create a baseline snapshot of all shared/critical files BEFORE creating worktrees. This snapshot will be compared against each worktree's state after implementation (step 9) to detect unintended mutations to shared files. This is critical for parallel safety.

---

## MANDATORY EXECUTION RULES

- 🛑 This snapshot MUST complete before ANY worktree is created
- 📖 Capture EXACT content — checksums and/or full content
- 🚫 NEVER skip files that appear in multiple story scopes
- 🎯 SEQUENTIAL execution — orchestrator only

---

## EXECUTION SEQUENCE

### 1. Identify Shared Files to Snapshot

Build the shared file list from two sources:

**A. Known Critical Files (always snapshot):**

```
# Package management
package.json
package-lock.json (or yarn.lock / pnpm-lock.yaml)
build.gradle / build.gradle.kts
pom.xml
pyproject.toml / requirements.txt

# Configuration
tsconfig.json / tsconfig.*.json
.eslintrc* / eslint.config.*
prettier.config.* / .prettierrc*
jest.config.* / vitest.config.*
webpack.config.* / vite.config.*
tailwind.config.*

# Schema / Type files
**/schema.prisma
**/types/index.ts (or equivalent shared type barrel)
**/api/types.ts
**/*.graphql (schema files)
**/openapi.yaml / **/openapi.json

# Other shared
.env.example
docker-compose.yml
Dockerfile
Makefile
```

**B. Cross-Story Files (from step 1 analysis):**

Use `{shared_file_candidates}` from step 1 — files identified as being in scope for multiple stories.

### 2. Capture Snapshot

For each identified shared file that EXISTS in the repository:

```bash
# Get content hash for each file
git hash-object {file_path}
```

Store the snapshot as a mapping:

```yaml
shared_file_snapshot:
  timestamp: "{current_timestamp}"
  base_branch: "{base_branch}"
  base_commit: "{result of: git rev-parse HEAD}"
  files:
    - path: "package.json"
      hash: "{git hash-object result}"
      exists: true
    - path: "src/types/index.ts"
      hash: "{git hash-object result}"
      exists: true
    - path: "prisma/schema.prisma"
      hash: null
      exists: false  # File doesn't exist, track creation
```

### 3. Capture Dependency Lock State

Special handling for lock files — these are the most common source of merge conflicts:

```bash
# Record exact state of lock files
git hash-object package-lock.json 2>/dev/null || echo "NOT_FOUND"
git hash-object yarn.lock 2>/dev/null || echo "NOT_FOUND"
git hash-object pnpm-lock.yaml 2>/dev/null || echo "NOT_FOUND"
```

**Flag if any story's scope includes dependency changes:**
- If YES: warn user that parallel dependency changes will likely conflict
- Record which stories plan to modify dependencies

### 4. Present Snapshot Summary to User

```
📸 SHARED FILE SNAPSHOT
═══════════════════════════════════════

Base commit: {short_hash} on {base_branch}
Timestamp: {timestamp}
Files tracked: {count}

Critical files snapshotted:
  ✅ package.json (hash: {short_hash})
  ✅ tsconfig.json (hash: {short_hash})
  ⬜ prisma/schema.prisma (not found — will track creation)
  ...

Cross-story overlap files:
  ⚠️ src/types/index.ts — touched by stories: {story-1}, {story-3}
  ⚠️ src/api/routes.ts — touched by stories: {story-2}, {story-4}

{if dependency_changes_planned}
⚠️ WARNING: Stories {ids} plan to modify dependencies.
   Parallel dependency changes may cause lock file conflicts.
   Consider implementing these stories sequentially.
{/if}
```

### 5. Store Snapshot State

Persist the complete snapshot for use in step 9 (Shared File Mutation Check):

```yaml
snapshot_state:
  shared_file_snapshot: {full snapshot from above}
  dependency_warning: boolean
  cross_story_overlap_files: list[{path, stories}]
```

### 6. Proceed to Worktree Setup

```
✅ Shared file snapshot captured ({count} files).
   This baseline will be used after implementation to detect mutations.
   Loading Step 3: Worktree Setup...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ All known critical files checked for existence and hashed
- ✅ Cross-story overlap files from step 1 included
- ✅ Lock file state explicitly captured
- ✅ Dependency change warnings issued if applicable
- ✅ Complete snapshot persisted for step 9

## FAILURE MODES

- ❌ Not capturing lock file state (most common conflict source)
- ❌ Missing cross-story overlap files
- ❌ Not warning about parallel dependency changes
- ❌ Snapshot taken AFTER worktree creation (invalidates baseline)
