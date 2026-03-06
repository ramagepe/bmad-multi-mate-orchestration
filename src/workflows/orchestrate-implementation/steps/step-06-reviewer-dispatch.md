---
name: step-06-reviewer-dispatch
description: "Invoke reviewer sub-agents for code review of each implementation"
nextStepFile: './step-07-corrective-loop.md'
phase: sequential-dispatch
phase_number: 3
executor: sub-agents
skipToStepFile: './step-08-cross-validation.md'
skipCondition: "all stories approved — no corrections needed"
---

# Step 6: Reviewer Dispatch

**Progress: Step 6 of 14** — Next: Corrective Loop
**Phase:** 3 (SEQUENTIAL DISPATCH — Reviewers for each story)

---

## STEP GOAL

Dispatch reviewer sub-agents to perform code review on each story that passed output collection. Reviewers validate acceptance criteria, code quality, test coverage, and flag security or shared file concerns.

---

## MANDATORY EXECUTION RULES

- 🛑 Only review stories classified as `READY_FOR_REVIEW` from step 5
- 📖 Each reviewer gets the FULL story file + diff of changes
- 🚫 Reviewers are READ-ONLY — they must NOT modify code
- 🎯 One reviewer per story, dispatched via Task tool

---

## SUB-AGENT CONTRACT: REVIEWER

Each Task tool invocation MUST include this complete contract:

```markdown
## Reviewer Sub-Agent Contract

**Role:** Code Reviewer
**Story:** {story_file_path}
**Worktree:** {worktree_path}
**Branch:** {branch_name}

### Your Working Directory
You are reviewing code in: {worktree_path}
You have READ-ONLY access. Do NOT modify any files.

### CRITICAL RESTRICTIONS
- ✅ You MAY: read any file in the worktree
- ✅ You MAY: run tests and lint commands
- ✅ You MAY: run `git diff` and `git log` to understand changes
- 🛑 You MUST NOT: edit, create, or delete any files
- 🛑 You MUST NOT: run `git add`, `git commit`, `git push`, or any modifying git command
- 🛑 You MUST NOT: install packages or modify system state

### What to Review
1. Read the story file completely: {story_file_path}
2. Review the diff: `git diff {base_branch}..HEAD`
3. Read all changed files in full context
4. Run the project's test suite
5. Run the project's linter

### Review Checklist — Evaluate EACH item
1. **Acceptance Criteria Met**: Every AC in the story file is implemented and verifiable
2. **Code Quality**: Code follows project conventions, is readable, maintainable
3. **Test Coverage**: New functionality has adequate tests, edge cases covered
4. **No Unintended Side Effects**: Changes don't break existing functionality
5. **Shared File Safety**: No unintended modifications to shared/common files
6. **Security**: No vulnerabilities introduced (hardcoded secrets, SQL injection, XSS, etc.)
7. **Architecture Compliance**: Implementation follows the architecture defined in the story
8. **Error Handling**: Proper error handling for edge cases and failure modes

### Reporting Format — You MUST end with this exact structure

```yaml
review_report:
  story_id: "{story-id}"
  status: "approved"  # or "needs_changes" or "rejected"
  overall_quality: "high"  # high | medium | low
  acceptance_criteria:
    - criterion: "AC description from story"
      met: true/false
      evidence: "Where/how it's implemented"
  findings:
    - severity: "critical"  # critical | major | minor | suggestion
      category: "security"  # security | quality | testing | architecture | shared_files
      description: "What the issue is"
      file: "path/to/file"
      line: 42
      suggestion: "How to fix it"
  test_results:
    tests_run: true/false
    tests_passed: true/false
    coverage_assessment: "adequate"  # adequate | insufficient | none
  shared_file_concerns:
    - file: "path/to/shared/file"
      change_type: "modified"  # modified | created | deleted
      concern: "Description of concern"
  suggested_improvements:
    - "Improvement suggestion 1"
    - "Improvement suggestion 2"
  summary: "Overall review summary"
```

### Review Standards
- **approved**: All ACs met, no critical/major findings, tests pass
- **needs_changes**: ACs mostly met but has critical/major findings that must be fixed
- **rejected**: Fundamental issues — wrong approach, missing core ACs, security vulnerabilities
```

---

## EXECUTION SEQUENCE

### 1. Prepare Review Queue

From `{stories_for_review}`:

```yaml
review_queue:
  - story_id: "{story-id-1}"
    story_file: "{path}"
    worktree_path: "{worktree_base_path}/{story-id-1}"
    branch: "{story-id-1}"
    implementation_summary: "{from collection step}"
```

### 2. Generate Diffs for Context

For each story, pre-generate the diff to include in the review context:

```bash
git -C {worktree_path} diff {base_branch}..HEAD
git -C {worktree_path} log --oneline {base_branch}..HEAD
```

### 3. Dispatch Reviewers

For EACH story in the review queue, dispatch a reviewer via Task tool:

```
Task tool invocation:
  - prompt: {Complete reviewer contract with all placeholders resolved, including the diff output}
  - workdir: {worktree_path}
```

**Concurrency:** Reviewers can run in parallel (up to `{max_parallel_agents}`), since they are read-only.

### 3.5 Handle Reviewer Failures

IF a reviewer sub-agent fails (no output, error, or timeout):

- Mark the story as `review_failed`
- Present to user:
  ```
  ⚠️ Reviewer failed for story {story-id}
  Error: {error details or "no output received"}
  
  Options:
  [R] Retry review (dispatch new reviewer)
  [S] Skip review — treat as approved (at your risk)
  [M] Manual review — I'll review it myself
  ```

- **R**: Re-dispatch reviewer (max 2 retries)
- **S**: Add to stories_approved with flag `review_skipped: true`
- **M**: User reviews, then tells orchestrator approved/needs_changes

### 4. Collect Review Results

As each reviewer completes, capture the `review_report`:

```yaml
review_results:
  - story_id: "{story-id-1}"
    status: "approved"
    report: {parsed review_report}
  - story_id: "{story-id-2}"
    status: "needs_changes"
    report: {parsed review_report}
```

### 5. Classify Review Outcomes

| Review Status | Next Action |
|--------------|-------------|
| **approved** | Move to cross-validation (step 8) |
| **needs_changes** | Enter corrective loop (step 7) |
| **rejected** | Escalate to user for decision |

### 6. Present Review Summary

```
🔍 CODE REVIEW RESULTS
═══════════════════════════════════════

  ✅ {story-id-1}: APPROVED — High quality, all ACs met
  ⚠️ {story-id-2}: NEEDS CHANGES — 2 critical findings
     - {finding_1_summary}
     - {finding_2_summary}
  ❌ {story-id-3}: REJECTED — Wrong architectural approach
     - {rejection_reason}

Summary: {approved_count} approved, {changes_count} need changes, {rejected_count} rejected
```

### 7. Handle Rejections

For rejected stories, present to user:

```
Story {story-id} was REJECTED by reviewer.
Reason: {rejection_details}

Options:
[F] Force approve (override reviewer)
[R] Re-implement (re-dispatch implementor with reviewer feedback)
[D] Drop story from this run
```

### 8. Store Review State

```yaml
review_state:
  review_results: {from above}
  stories_approved: list[string]
  stories_needs_changes: list[{story_id, findings}]
  stories_rejected: list[{story_id, reason, user_decision}]
```

### 9. Route to Next Step

- **IF any stories need changes** → Load `{nextStepFile}` (step-07 — Corrective Loop)
- **IF all stories approved (or user overrode)** → Load `{skipToStepFile}` (step-08 — Cross-Validation)

```
{if needs_changes}
⚠️ {count} stories need corrections.
   Loading Step 7: Corrective Loop...
{else}
✅ All stories approved by reviewers!
   Skipping corrective loop — Loading Step 8: Cross-Validation...
{/if}
```

Load, read completely, then execute the appropriate next step file.

---

## SUCCESS METRICS

- ✅ Every ready story received a code review
- ✅ Review reports parsed and classified
- ✅ Findings clearly presented to user
- ✅ Rejected stories escalated with options
- ✅ Clean routing to corrective loop or cross-validation

## FAILURE MODES

- ❌ Reviewers modifying code (must be read-only)
- ❌ Not including the diff in reviewer context
- ❌ Auto-approving without actual review
- ❌ Not escalating rejected stories to user
