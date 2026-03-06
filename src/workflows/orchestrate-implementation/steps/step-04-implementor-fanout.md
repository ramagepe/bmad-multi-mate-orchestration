---
name: step-04-implementor-fanout
description: "Dispatch implementor sub-agents with contracts — PARALLEL, one per worktree"
nextStepFile: './step-05-output-collection.md'
phase: parallel
phase_number: 2
executor: sub-agents
max_parallel: "{max_parallel_agents}"
security: critical
---

# Step 4: Implementor Fan-Out (Parallel)

**Progress: Step 4 of 14** — Next: Output Collection
**Phase:** 2 (PARALLEL — Sub-Agent Dispatch)
**Concurrency:** Up to `{max_parallel_agents}` simultaneous sub-agents

---

## STEP GOAL

Dispatch one implementor sub-agent per story using the Task tool. Each sub-agent receives a strict contract defining their worktree, scope boundaries, and exit criteria. Sub-agents work in PARALLEL within the configured concurrency limit.

---

## MANDATORY EXECUTION RULES

- 🛑 **NEVER** dispatch more than `{max_parallel_agents}` simultaneously
- 📖 Each sub-agent gets its OWN Task tool invocation with full contract
- 🚫 Sub-agents must NEVER: create branches, fetch, pull, push, or operate outside their worktree
- 🎯 Use the Task tool to dispatch each sub-agent
- ⏱️ Each sub-agent has a timeout of `{sub_agent_timeout_minutes}` minutes

---

## SUB-AGENT CONTRACT: IMPLEMENTOR

Each Task tool invocation MUST include this complete contract as the prompt/instructions:

```markdown
## Implementor Sub-Agent Contract

**Role:** Story Implementor
**Story:** {story_file_path}
**Worktree:** {worktree_path}
**Branch:** {branch_name} (PRE-CREATED — do NOT create or modify branches)

### Your Working Directory
You MUST work exclusively in: {worktree_path}
All file operations, all git commands — everything happens in this directory.

### CRITICAL RESTRICTIONS — VIOLATION IS SYSTEM FAILURE
- ✅ You MAY: edit files, create files, delete files within {worktree_path}
- ✅ You MAY: `git add` and `git commit` within {worktree_path}
- 🛑 You MUST NOT: `git branch`, `git checkout`, `git fetch`, `git pull`, `git push`
- 🛑 You MUST NOT: modify files outside {worktree_path}
- 🛑 You MUST NOT: install global packages or modify system state

### Scope Boundaries
**Writable paths (relative to worktree root):**
{writable_paths from story scope — e.g., src/features/story-feature/, tests/features/story-feature/}

**Read-only reference paths:**
{readonly_paths — e.g., src/types/, src/shared/, docs/architecture/}

### Your Mission
1. Read the story file completely: {story_file_path}
2. Understand ALL acceptance criteria
3. Implement the story following the architecture defined in the story
4. Write tests for all new functionality
5. Ensure all existing tests still pass
6. Commit your work with a descriptive message

### Exit Criteria — ALL must be met
- All acceptance criteria from the story file are implemented
- All existing tests pass (run the project's test command)
- New tests written for new functionality
- Code committed to branch with descriptive commit message
- No lint errors (run the project's lint command)

### Reporting Format — You MUST end with this exact structure
When you complete (or fail), output this report:

```yaml
implementation_report:
  story_id: "{story-id}"
  status: "completed"  # or "failed" or "blocked"
  files_changed:
    - path/to/file1.ts
    - path/to/file2.ts
  files_created:
    - path/to/new-file.ts
  commit_hash: "{hash from git log -1 --format=%H}"
  commit_message: "{your commit message}"
  tests_run: true/false
  tests_passed: true/false
  lint_clean: true/false
  summary: "Brief description of what was implemented"
  issues_encountered:
    - "Any blockers or concerns"
  time_spent_estimate: "approximate time"
```

### Epic Context
{epic_context from step 1 — brief summary so implementor understands the bigger picture}
```

---

## EXECUTION SEQUENCE

### 1. Build Dispatch Queue

From `{worktree_registry}`, create the dispatch queue:

```yaml
dispatch_queue:
  - story_id: "{story-id-1}"
    story_file: "{path}"
    worktree_path: "{worktree_base_path}/{story-id-1}"
    branch: "{story-id-1}"
    status: pending
  - story_id: "{story-id-2}"
    # ...
```

### 2. Dispatch Sub-Agents with Concurrency Control

**Dispatch pattern — BATCH MODE:**

The Task tool does not support "dispatch N, wait for first to complete, dispatch next".
Use BATCH dispatch instead:

```
total_stories = stories.count
batch_size = min(max_parallel_agents, total_stories)
batch_number = 0

WHILE stories remain in queue:
  batch_number += 1
  current_batch = next {batch_size} stories from queue
  
  Dispatch ALL stories in current_batch simultaneously using Task tool
  (multiple Task tool calls in the same message)
  
  WAIT for ALL in current_batch to complete
  
  Record results for current_batch
  
  Continue to next batch
```

**Note:** This means if one sub-agent in a batch finishes in 5 minutes and another takes 25 minutes, the third slot is idle for 20 minutes. This is a known limitation of the Task tool — not a bug.

**For each dispatch, use the Task tool:**

```
Task tool invocation:
  - prompt: {Complete implementor contract from above, with all placeholders resolved}
  - workdir: {worktree_path}  # CRITICAL: set working directory to worktree
```

### 3. Monitor Dispatch Status

Track each sub-agent:

```yaml
dispatch_status:
  - story_id: "{story-id-1}"
    dispatch_time: "{timestamp}"
    status: "running"  # pending | running | completed | failed | timeout
    task_id: "{task tool reference}"
  - story_id: "{story-id-2}"
    # ...
```

### 4. Wait for Completion

Wait for all dispatched sub-agents to complete or timeout.

**Timeout handling:**
- If a sub-agent exceeds `{sub_agent_timeout_minutes}`:
  - Mark as `timeout` in dispatch status
  - Log the timeout event
  - Do NOT kill the sub-agent if it's still running — just note it

### 5. Collect Raw Results

As each sub-agent completes, capture its output (the `implementation_report` structure).

**Store in dispatch_results:**

```yaml
dispatch_results:
  - story_id: "{story-id-1}"
    status: "completed"
    raw_output: "{full sub-agent output}"
    report: {parsed implementation_report}
  - story_id: "{story-id-2}"
    status: "failed"
    raw_output: "{full sub-agent output}"
    report: {parsed implementation_report or null}
```

### 6. Quick Status to User

While collecting (or after all complete), show progress:

```
🔧 IMPLEMENTOR DISPATCH STATUS
═══════════════════════════════════════

  {story-id-1}: ✅ Completed — {commit_hash_short}
  {story-id-2}: ✅ Completed — {commit_hash_short}
  {story-id-3}: 🔄 Running...
  {story-id-4}: ⏳ Queued (waiting for slot)

Active: {running_count}/{max_parallel_agents}
Completed: {completed_count}/{total_count}
```

### 7. Proceed to Output Collection

Once all sub-agents have finished (completed, failed, or timed out):

```
✅ All implementors finished.
   Completed: {count}, Failed: {count}, Timeout: {count}
   Loading Step 5: Output Collection...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SECURITY RULES ENFORCED

1. ✅ Each sub-agent gets workdir set to its specific worktree
2. ✅ Contract explicitly forbids branch operations
3. ✅ Contract explicitly forbids push/fetch/pull
4. ✅ Scope boundaries defined per story
5. ✅ Sub-agents cannot see or access other worktrees

---

## SUCCESS METRICS

- ✅ All stories dispatched within concurrency limits
- ✅ Each sub-agent received complete contract with correct worktree path
- ✅ Working directory correctly set for each Task tool invocation
- ✅ All sub-agent outputs captured
- ✅ Dispatch status tracked for every story

## FAILURE MODES

- ❌ Dispatching more sub-agents than `{max_parallel_agents}`
- ❌ Forgetting to set workdir on Task tool invocation
- ❌ Not including the full contract in the Task prompt
- ❌ Not capturing sub-agent output for failed/timed-out agents
- ❌ Not resolving placeholders in the contract before dispatch
