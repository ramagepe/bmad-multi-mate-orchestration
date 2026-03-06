---
name: orchestrate-implementation
description: "Fan-out with git worktrees + code review + quality gates + merge gate. Implementation mode with full isolation."
module: bmo
installed_path: '{project-root}/_bmad/bmo/workflows/orchestrate-implementation'
status: implemented
firstStep: './steps/step-01-readiness-check.md'
---

# Orchestrate Implementation Workflow

**Module:** bmo
**Status:** Implemented — 14 step files in `./steps/`
**Mode:** Implementation (git worktrees mandatory)
**Executor:** Tito 🧉 (Orchestrator Agent)

---

## Workflow Overview

**Goal:** Orchestrate parallel story implementation across isolated git worktrees with automated code review, quality gates, and human-controlled merge gate.

**Description:** The orchestrator creates worktrees and branches sequentially (Phase 1), dispatches implementor sub-agents in parallel (Phase 2), then collects results, runs cross-validation, and presents merge candidates to the user (Phase 3). Includes corrective loops for code review feedback and shared file mutation detection.

**Trigger:** User selects [OD] from orchestrator menu or requests implementation orchestration.

---

## Input Contract

```yaml
input:
  epic_path: string          # Path to epic file or epic ID
  stories: list[string]      # List of story file paths or story IDs
  mode: implementation        # Fixed for this workflow
  parallel_limit: int        # Max concurrent sub-agents (from config)
  base_branch: string        # Branch to create worktrees from
```

---

## Configuration (from bmo/config.yaml)

```yaml
max_parallel_agents: 3
max_correction_loops: 3
sub_agent_timeout_minutes: 30
worktree_base_path: "{project-root}/.claude/worktrees"
```

---

## Step Files

| Step | File | Name | Phase | Executor |
|------|------|------|-------|----------|
| 1 | `step-01-readiness-check.md` | Readiness Check | Sequential (Phase 1) | Orchestrator |
| 2 | `step-02-shared-file-snapshot.md` | Shared File Snapshot | Sequential (Phase 1) | Orchestrator |
| 3 | `step-03-worktree-setup.md` | Worktree Setup | Sequential (Phase 1) | Orchestrator |
| 4 | `step-04-implementor-fanout.md` | Implementor Fan-Out | Parallel (Phase 2) | Sub-Agents |
| 5 | `step-05-output-collection.md` | Output Collection | Sequential (Phase 3) | Orchestrator |
| 6 | `step-06-reviewer-dispatch.md` | Reviewer Dispatch | Sequential Dispatch (Phase 3) | Sub-Agents |
| 7 | `step-07-corrective-loop.md` | Corrective Loop | Corrective (Phase 3) | Mixed |
| 8 | `step-08-cross-validation.md` | Cross-Validation | Sequential (Phase 3) | Sub-Agent |
| 9 | `step-09-shared-file-mutation-check.md` | Shared File Mutation Check | Sequential (Phase 3) | Orchestrator |
| 10 | `step-10-pre-merge-gate.md` | Pre-Merge Gate | Sequential (Phase 3) | Orchestrator |
| 11 | `step-11-human-approval-gate.md` | Human Approval Gate | Sequential (Phase 3) | Orchestrator |
| 12 | `step-12-push-approved-branches.md` | Push Approved Branches | Sequential (Phase 3) | Orchestrator |
| 13 | `step-13-cleanup.md` | Cleanup | Sequential (Phase 3) | Orchestrator |
| 14 | `step-14-summary-report.md` | Summary Report | Sequential (Phase 3) | Orchestrator |

---

## Execution Phases

### Phase 1: Preparation (Steps 1-3) — SEQUENTIAL

```
Step 1: Validate stories → readiness classification
Step 2: Snapshot shared files → baseline for mutation detection
Step 3: Create worktrees/branches → one per story, sequential
```

All orchestrator-only. No sub-agents. Must complete before Phase 2.

### Phase 2: Implementation (Step 4) — PARALLEL

```
Step 4: Dispatch implementors → one sub-agent per worktree
        Concurrency limited to max_parallel_agents
        Sub-agents: edit, add, commit — NOTHING ELSE
```

### Phase 3: Review, Validation & Delivery (Steps 5-14) — SEQUENTIAL

```
Step 5:  Collect outputs → verify commits, classify results
Step 6:  Dispatch reviewers → code review per story
Step 7:  Corrective loop → fix review findings (max 3 iterations)
Step 8:  Cross-validate → check coherence across stories
Step 9:  Mutation check → diff worktrees against snapshot
Step 10: Pre-merge gate → conflict detection
Step 11: Human approval → per-branch user approval
Step 12: Push → only approved branches
Step 13: Cleanup → remove worktrees
Step 14: Summary → execution report
```

---

## Git Race Condition Prevention

```
Phase 1 (SEQUENTIAL - orchestrator only):
  for each story:
    git worktree add {worktree_base_path}/{story-id} -b {story-id}
  → All worktrees and branches exist before any sub-agent starts

Phase 2 (PARALLEL - sub-agents):
  Each sub-agent receives pre-created worktree path
  Each sub-agent only does: edit files, git add, git commit
  NO branch creation, NO fetch, NO pull, NO push

Phase 3 (SEQUENTIAL - orchestrator only):
  Collect results
  Run cross-validation
  Present to user for approval
  Push only approved branches (with user confirmation)
```

---

## Quality Gates

| Gate | Step | Validates | Failure Action |
|------|------|-----------|----------------|
| Readiness Check | 1 | Stories implementation-ready | Flag incomplete, proceed with ready |
| Post-Implementation | 5 | Tests pass, lint clean, commit exists | Enter corrective loop |
| Post-Review | 6-7 | Reviewer approved, no critical findings | Enter corrective loop or escalate |
| Cross-Validation | 8 | No conflicts across stories | Alert user, let them decide |
| Pre-Merge | 10 | No git conflicts, shared mutations flagged | Alert user with conflict details |
| Pre-Merge Tests | 10 | Tests pass in all worktrees | Mark as MERGE_BLOCKED, user must override |
| Human Approval | 11 | User explicitly approves each branch | Skip unapproved branches |

---

## Sub-Agent Contracts

### Implementor (Steps 4, 7)

```yaml
role: implementor
worktree_path: string        # Absolute path to pre-created worktree
story_file_path: string      # Full story spec
branch_name: string          # Pre-created by orchestrator
scope_boundary:
  writable_paths: list[string]
  readonly_paths: list[string]
exit_criteria:
  - "All acceptance criteria implemented"
  - "All existing tests pass"
  - "New tests written for new functionality"
  - "Code committed with descriptive message"
reporting: {status, files_changed, commit_hash, summary, issues}
```

### Reviewer (Steps 6, 7)

```yaml
role: reviewer
worktree_path: string        # Read-only access
story_file_path: string
review_checklist:
  - "Acceptance criteria met"
  - "Code quality and standards"
  - "Test coverage adequate"
  - "No shared file side effects"
  - "No security vulnerabilities"
reporting: {status, findings, suggested_improvements}
```

### Cross-Validator (Step 8)

```yaml
role: cross_validator
worktree_paths: list[string]  # All approved worktrees
epic_file: string
validation_checklist:
  - "No duplicate work"
  - "No conflicting business rules"
  - "Shared interfaces consistent"
  - "No coverage gaps"
  - "Import/dependency consistency"
reporting: {status, conflicts, gaps, recommendations}
```

---

## Security Rules — NON-NEGOTIABLE

1. Commits ONLY in assigned worktrees, NEVER on main/base branch
2. Push EXCLUSIVELY with explicit user confirmation (Step 11)
3. All worktrees/branches created BEFORE fan-out (Step 3, sequential)
4. Sub-agents NEVER create branches — orchestrator only
5. Shared file mutations detected post-hoc and flagged to user (Step 9)
6. Force push NEVER executed without explicit user confirmation AND typing "FORCE"

---

## Output Contract

```yaml
output:
  execution_summary:
    stories_completed: list
    stories_failed: list
    stories_pending_review: list
  branches_created: list[string]
  branches_pushed: list[string]
  actions_pending_approval: list
  shared_file_mutations: list
  correction_loops_executed: int
  test_metrics:
    total_verifications: int
    independently_passed: int
    discrepancies_detected: int
    pre_merge_regressions: int
```

---

## Integration Points

- **BMM**: dev-story (invoked by implementor sub-agents), code-review (invoked by reviewer sub-agents)
- **Core**: Task tool (sub-agent dispatch mechanism)
- **TEA**: testarch-automate (post-implementation test generation), testarch-test-review (review phase)

**Note:** TEA module integration (testarch-automate, testarch-test-review) is documented 
as a v2 enhancement. v1 uses independent test execution by the orchestrator (Steps 5 and 10).

---

## Corrective Loop with Circuit Breaker

```
max_correction_loops: 3 (from config)

LOOP per story:
  1. Reviewer returns needs_changes
  2. Orchestrator re-dispatches implementor with review feedback
  3. Implementor applies fixes + commits
  4. Reviewer re-reviews
  5. IF approved → exit loop
  6. IF needs_changes AND loop_count < max → repeat
  7. IF needs_changes AND loop_count >= max → ESCALATE
     - Present findings and history to user
     - User decides: force approve, manual fix, or drop story
```

---

## Recovery Reference

For cleanup and recovery, reference these patterns:

```bash
git worktree list           # Scan existing worktrees
git worktree remove <path>  # Remove worktree
git worktree prune          # Clean stale references
git branch -d <branch>      # Safe delete (merged only)
git branch -D <branch>      # Force delete (user confirmed)
```

---

_Spec created on 2026-03-04 via BMAD Module workflow_
_Implemented on 2026-03-05 via create-workflow [C]onvert mode_
