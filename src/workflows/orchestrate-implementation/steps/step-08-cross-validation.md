---
name: step-08-cross-validation
description: "Check for conflicts, duplicate work, shared interface consistency across stories"
nextStepFile: './step-09-shared-file-mutation-check.md'
phase: sequential
phase_number: 3
executor: sub-agent
quality_gate: cross-validation
---

# Step 8: Cross-Validation

**Progress: Step 8 of 14** — Next: Shared File Mutation Check
**Phase:** 3 (SEQUENTIAL — Cross-Story Coherence Check)

---

## STEP GOAL

Validate that all implemented stories are coherent as a set: no duplicate work, no conflicting business rules, consistent shared interfaces, and no gaps in acceptance criteria coverage. This is a holistic check that individual reviewers cannot perform.

---

## MANDATORY EXECUTION RULES

- 🛑 This step examines ALL approved stories together — not individually
- 📖 The cross-validator needs access to ALL worktrees simultaneously
- 🚫 Cross-validator is READ-ONLY — no modifications to any worktree
- 🎯 Dispatch as a single sub-agent with multi-worktree context

---

## SUB-AGENT CONTRACT: CROSS-VALIDATOR

Dispatch via Task tool with this contract:

```markdown
## Cross-Validator Sub-Agent Contract

**Role:** Cross-Story Coherence Validator
**Epic:** {epic_path}
**Stories Under Validation:** {count} stories

### Worktrees to Examine (READ-ONLY)

{For each approved story:}
- **{story-id}**: {worktree_path}
  Story file: {story_file_path}
  Branch: {branch_name}

### Review Context (from steps 6-7)

{For each story:}
- **{story-id}**: 
  Review status: {approved / force_approved / review_skipped}
  Correction loops: {count}
  Known remaining issues: {list from force_approval or review findings}

**Use this context to:**
- Pay extra attention to force-approved stories
- Check if corrections in one story created inconsistencies with others
- Verify that known issues don't compound across stories

### CRITICAL RESTRICTIONS
- ✅ You MAY: read files in ALL worktrees listed above
- ✅ You MAY: run `git diff` to compare worktrees against base
- 🛑 You MUST NOT: modify any files in any worktree
- 🛑 You MUST NOT: run any modifying git commands
- 🛑 SUBAGENT-STOP: You are a SUB-AGENT dispatched for a specific task.
  Do NOT invoke BMO orchestration workflows or BMAD agent menus.
  Do NOT dispatch your own sub-agents via Task tool.
  Complete YOUR assigned task and report back. Nothing else.

### Your Mission

Examine all implemented stories as a COHESIVE SET and validate:

1. **No Duplicate Work**: Different stories haven't implemented the same functionality
2. **No Conflicting Business Rules**: Story A doesn't contradict Story B's logic
3. **Shared Interface Consistency**:
   - API contracts (routes, request/response shapes) are compatible
   - TypeScript/type definitions used consistently
   - Database schema changes are compatible
   - Shared utility functions aren't modified in conflicting ways
4. **No Coverage Gaps**: The epic's requirements are fully covered by the combined stories
5. **Import/Dependency Consistency**:
   - No conflicting package versions
   - Import paths are consistent
   - No circular dependency introduction

### How to Examine

For each pair of stories, compare:
```bash
# Files changed by story A
git -C {worktree_A} diff --name-only {base_branch}..HEAD

# Files changed by story B
git -C {worktree_B} diff --name-only {base_branch}..HEAD

# Find overlapping files
# For each overlap: compare the actual changes
```

For shared interfaces:
```bash
# Compare type definitions across worktrees
# Compare API route definitions
# Compare schema files
```

### Reporting Format

```yaml
cross_validation_report:
  status: "coherent"  # or "issues_found"
  stories_validated: list[string]
  conflicts:
    - story_a: "{story-id-a}"
      story_b: "{story-id-b}"
      description: "Both stories modify the User model differently"
      severity: "critical"  # critical | major | minor
      files_involved:
        - path: "src/models/User.ts"
          story_a_change: "Adds email field"
          story_b_change: "Renames name to fullName"
      recommendation: "Resolve before merging — manual merge needed"
  gaps:
    - description: "Epic requires feature X but no story implements it"
      affected_stories: list[string]
      severity: "major"
  duplicate_work:
    - description: "Both stories implement input validation utility"
      stories: list[string]
      recommendation: "Keep one implementation, remove the other"
  interface_inconsistencies:
    - description: "Story A exports UserDTO with `name`, Story B imports UserDTO expecting `fullName`"
      stories: list[string]
      files: list[string]
  dependency_conflicts:
    - description: "Story A adds lodash@4.17.21, Story B adds lodash@4.17.20"
      stories: list[string]
  recommendations:
    - "Recommendation 1"
    - "Recommendation 2"
  summary: "Overall coherence assessment"
```
```

---

## EXECUTION SEQUENCE

> **Note:** IF step 7 was skipped (all stories approved in step 6), use default corrective_state: 0 corrections, all stories from review_state.stories_approved. No force-approved or dropped stories.

### 1. Prepare Cross-Validation Context

Gather all information the cross-validator needs:

```yaml
validation_context:
  epic_file: "{epic_path}"
  stories:
    - id: "{story-id-1}"
      worktree: "{path}"
      story_file: "{path}"
      files_changed: list[string]  # from step 5 collection
    - id: "{story-id-2}"
      # ...
```

### 2. Dispatch Cross-Validator

Single Task tool invocation with the complete contract above.

**Working directory:** Set to project root (cross-validator needs access to all worktrees).

### 3. Process Validation Report

Parse the `cross_validation_report` and classify issues:

| Issue Type | Severity | Action |
|-----------|----------|--------|
| Conflicts (critical) | 🔴 | Must resolve before merge |
| Conflicts (major) | 🟡 | Should resolve, user decides |
| Gaps | 🟡 | Informational — may be intentional |
| Duplicate work | 🟡 | User decides which to keep |
| Interface inconsistencies | 🔴 | Must resolve before merge |
| Dependency conflicts | 🟡 | Usually auto-resolvable |

### 4. Present Cross-Validation Results

```
🔀 CROSS-VALIDATION RESULTS
═══════════════════════════════════════

Overall: {status}

{if conflicts exist}
🔴 CONFLICTS DETECTED:
  • {story-a} ↔ {story-b}: {description}
    Files: {files}
    Recommendation: {recommendation}
{/if}

{if gaps exist}
🟡 COVERAGE GAPS:
  • {gap_description}
    Affected: {stories}
{/if}

{if duplicate_work exists}
🟡 DUPLICATE WORK:
  • {description}
    Stories: {stories}
{/if}

{if interface_inconsistencies exist}
🔴 INTERFACE INCONSISTENCIES:
  • {description}
    Stories: {stories}
{/if}

{if no issues}
✅ All stories are coherent — no conflicts, gaps, or duplicates detected.
{/if}
```

### 5. Handle Critical Issues

If critical conflicts or interface inconsistencies found:

```
⚠️ Critical cross-story issues require resolution before merging.

Options:
[M] Manual resolution — I'll fix the conflicts
[R] Re-implement affected stories with conflict guidance
[F] Force proceed — I understand the risks
[X] Abort
```

- **M**: User resolves manually in worktrees, then re-run cross-validation
- **R**: Re-dispatch implementors for affected stories with conflict context
- **F**: Proceed with warning flags
- **X**: Abort workflow

### 6. Store Cross-Validation State

```yaml
cross_validation_state:
  report: {complete cross_validation_report}
  critical_issues_resolved: boolean
  user_overrides: list[string]  # Issues user chose to force-proceed on
```

### 7. Proceed to Shared File Mutation Check

```
✅ Cross-validation complete.
   Loading Step 9: Shared File Mutation Check...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ All story pairs examined for conflicts
- ✅ Shared interfaces validated for consistency
- ✅ Coverage gaps identified
- ✅ Critical issues escalated to user
- ✅ User resolved or acknowledged all critical issues

## FAILURE MODES

- ❌ Only comparing adjacent stories (must check ALL pairs)
- ❌ Not examining shared type/interface files
- ❌ Silently proceeding past critical conflicts
- ❌ Cross-validator modifying worktree files
