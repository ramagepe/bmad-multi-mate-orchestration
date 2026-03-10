---
name: step-03-sub-agent-dispatch
description: "Dispatch document processor sub-agents with contracts — PARALLEL, one per story"
nextStepFile: './step-04-output-collection.md'
phase: parallel
phase_number: 2
executor: sub-agents
max_parallel: "{max_parallel_agents}"
---

# Step 3: Sub-Agent Dispatch (Parallel)

**Progress: Step 3 of 8** — Next: Output Collection
**Phase:** 2 (PARALLEL — Sub-Agent Dispatch)
**Concurrency:** Up to `{max_parallel_agents}` simultaneous sub-agents

---

## STEP GOAL

Dispatch one document processor sub-agent per story using the Task tool. Each sub-agent receives a strict contract defining their assigned BMAD workflow, input story, output path, and exit criteria. Sub-agents work in PARALLEL within the configured concurrency limit, processing batches in dependency order.

---

## MANDATORY EXECUTION RULES

- 🛑 **NEVER** dispatch more than `{max_parallel_agents}` simultaneously
- 📖 Each sub-agent gets its OWN Task tool invocation with full contract
- 🚫 Sub-agents must NEVER: modify original story files, perform git operations, or write outside their output path
- 🎯 Use the Task tool to dispatch each sub-agent
- ⏱️ Each sub-agent has a timeout of `{sub_agent_timeout_minutes}` minutes
- 📋 Respect batch dependency ordering from step 2
- 🔧 **TEMPLATE RESOLUTION:** Before dispatching each contract, you MUST resolve ALL `{if}`/`{/if}` conditionals — include ONLY the blocks matching the story's `task_type`, remove all others including the `{if}`/`{/if}` tags themselves. Also resolve ALL `{placeholder}` variables to their actual values. The sub-agent must receive a CLEAN contract with no unresolved conditionals or placeholders.

---

## SUB-AGENT CONTRACT: DOCUMENT PROCESSOR

Each Task tool invocation MUST include this complete contract as the prompt/instructions:

```markdown
## Document Processor Sub-Agent Contract

**Role:** Document Processor
**Task Type:** {task_type}
**Workflow to Invoke:** {workflow_to_invoke}
**Story Key:** {story_key}
{if task_type == "story_creation"}
**Epic File:** {epic_path}
**Output File:** {stories_output_path}/{story_key}.md
{/if}
{if task_type != "story_creation"}
**Story File:** {story_file_path}
**Output Path:** {stories_output_path}
{/if}

### Your Mission

{if task_type == "story_creation"}
Load and execute completely: {installed_path}/steps/story-creation-panel.md
That document is your complete program — follow it top to bottom. It handles
context extraction, story generation, and multi-perspective validation.
Write the output story file to: {stories_output_path}/{story_key}.md
{/if}

{if task_type == "story_refinement"}
Load and execute completely: {installed_path}/steps/story-refinement-panel.md
That document is your complete program — follow it top to bottom. It handles
multi-perspective analysis, synthesis, and optimized story output.
Write the refined story to: {stories_output_path}/{story_key}.md
{/if}

{if task_type != "story_creation" AND task_type != "story_refinement"}
1. Read the story file completely: {story_file_path}
2. Understand ALL acceptance criteria
3. Execute the BMAD workflow: {workflow_to_invoke}
4. Produce document artifacts that satisfy the story's acceptance criteria
5. Write all artifacts to: {stories_output_path}
{/if}

### Epic Context
{epic_context from step 1 — summary so processor understands the bigger picture}

### CRITICAL RESTRICTIONS — VIOLATION IS SYSTEM FAILURE
- ✅ You MAY: read any file in the project for context
- ✅ You MAY: create and write files in {stories_output_path}
- ✅ You MAY: read the epic file ({epic_path})
- 🛑 You MUST NOT: modify the epic file ({epic_path})
{if task_type != "story_creation"}
- ✅ You MAY: read the original story file ({story_file_path})
- 🛑 You MUST NOT: modify the original story file ({story_file_path}) — write the refined version to {stories_output_path}/{story_key}.md instead
{/if}
- 🛑 You MUST NOT: write files outside {stories_output_path} (except reading for context)
- 🛑 You MUST NOT: perform any git operations (add, commit, push, branch, etc.)
- 🛑 You MUST NOT: install packages or modify system state
- 🛑 SUBAGENT-STOP: You are a SUB-AGENT dispatched for a specific task.
  Do NOT invoke BMO orchestration workflows or BMAD agent menus.
  Do NOT dispatch your own sub-agents via Task tool.
  Complete YOUR assigned task and report back. Nothing else.

### Workflow Execution
Invoke the **{workflow_to_invoke}** workflow with the story as input.
Follow that workflow's steps and produce the artifacts it defines.

{if task_type == "story_creation"}
### Panel Inputs
- Epic file: {epic_path} — locate story {story_key} (Epic {epic_num}, Story {story_num})
- Architecture document: {architecture_path} (read if path provided)
- PRD: {prd_path} (read if path provided)
- Previous story in epic sequence: {previous_story_path} (read if applicable)
{/if}

{if task_type == "story_refinement"}
### Panel Inputs
- Story file: {story_file_path}
- Architecture document: {architecture_path} (read if path provided)
- Previous story in epic sequence: {previous_story_path} (read if applicable)
{/if}

{if task_type == "document_creation"}
- Create the requested document based on the story spec
- Output: document file(s) in {stories_output_path}
{/if}

{if task_type == "analysis"}
- Perform deep analysis / elicitation on the story's domain
- Output: analysis report in {stories_output_path}
{/if}

### Output Standards
- All artifacts must be in markdown format unless otherwise specified
- Use clear headings, structured sections, and proper formatting
- Include a summary section at the top of each artifact
- Reference the source story ID in each artifact

### Exit Criteria — ALL must be met
- All acceptance criteria are addressed in the output story file
- Output follows document standards (structured, clear, formatted)
- Story file written to {stories_output_path}/{story_key}.md
- No original/protected files modified (epic file, existing story files)

### Reporting Format — You MUST end with this exact structure
When you complete (or fail), output this report:

```yaml
processing_report:
  story_id: "{story-id}"
  status: "completed"  # or "failed" or "blocked"
  task_type: "{task_type}"
  workflow_invoked: "{workflow_to_invoke}"
  artifacts_produced:
    - path: "{stories_output_path}/{story_key}.md"
      type: "created_story"  # or "refined_story" or "spec" or "analysis" or "adr" etc.
      description: "Brief description"
  acceptance_criteria_coverage:
    - criterion: "AC description from story"
      addressed: true/false
      evidence: "Which artifact addresses this"
  summary: "Brief description of what was produced"
  issues_encountered:
    - "Any blockers or concerns"
  time_spent_estimate: "approximate time"
```
```

---

## EXECUTION SEQUENCE

### 1. Record Pre-Dispatch Baselines

Before any sub-agent dispatch, record baseline timestamps for contract violation detection (used in step 4):

```bash
# Record modification timestamps for original story files (refinement only)
for story in {stories_proceeding}:
  if story.story_file_exists:
    stat -c '%Y %n' {story.story_file_path}

# Always record the epic file timestamp (protected in both creation and refinement)
stat -c '%Y %n' {epic_path}
```

```yaml
baseline_timestamps:
  story_files:                              # Only for refinement tasks
    - file: "{story_file_path}"
      mtime: "{timestamp_before_dispatch}"
  protected_files:                          # Always
    - file: "{epic_path}"
      mtime: "{timestamp_before_dispatch}"
  dispatch_start_time: "{current_timestamp}"  # For step 8 report
```

### 2. Initialize Dispatch from Plan

From `{dispatch_plan}` (step 2), prepare the dispatch queue:

```yaml
dispatch_queue:
  - batch: 1
    stories:
      - story_id: "{story-id-1}"
        story_key: "{story_key}"
        story_file: "{path}"           # null for story_creation
        task_type: "{type}"
        workflow: "{workflow}"
        output_path: "{stories_output_path}"
        status: pending
      - story_id: "{story-id-2}"
        # ...
  - batch: 2
    wait_for: ["{story-id-1}"]
    stories:
      - story_id: "{story-id-4}"
        # ...
```

### 3. Dispatch Sub-Agents by Batch

**Dispatch pattern — BATCH MODE with dependency ordering:**

```
FOR each batch in dispatch_plan.batches (in order):
  
  IF batch.wait_for is defined:
    VERIFY all dependency stories have completed successfully
    IF any dependency failed:
      → Present to user: dependent stories cannot proceed
      → Options: [S] Skip dependent stories | [F] Force proceed (see warning) | [X] Abort
      
      ⚠️ WARNING for [F] Force proceed: The dependent stories will be processed 
      WITHOUT their upstream dependency's output. The sub-agent will receive the 
      original story spec but not the refined artifacts from {failed_dependency}. 
      This may result in inconsistent or incomplete artifacts that require 
      correction in the cross-validation phase.

  current_batch = batch.stories
  batch_size = min(max_parallel_agents, current_batch.count)
  
  Dispatch ALL stories in current_batch simultaneously using Task tool
  (multiple Task tool calls in the same message)
  
  WAIT for ALL in current_batch to complete
  
  Record results for current_batch
  
  Continue to next batch
```

**Handling skipped stories:** If user selects [S] to skip dependent stories, record them
in dispatch_results with `status: "skipped"` and `reason: "dependency_failed"` so they are
properly tracked through steps 4 and 8.

**Note:** This means if one sub-agent in a batch finishes in 5 minutes and another takes 25 minutes, the third slot is idle for 20 minutes. This is a known limitation of the Task tool — not a bug.

**For each dispatch, use the Task tool:**

```
Task tool invocation:
  - prompt: {Complete document processor contract from above, with all placeholders resolved}
```

### 4. Monitor Dispatch Status

Track each sub-agent:

```yaml
dispatch_status:
  - story_id: "{story-id-1}"
    batch: 1
    dispatch_time: "{timestamp}"
    status: "running"  # pending | running | completed | failed | timeout
    task_id: "{task tool reference}"
  - story_id: "{story-id-2}"
    # ...
```

### 5. Wait for Completion

Wait for all dispatched sub-agents in the current batch to complete or timeout.

**Timeout handling:**
- If a sub-agent exceeds `{sub_agent_timeout_minutes}`:
  - Mark as `timeout` in dispatch status
  - Log the timeout event
  - Do NOT kill the sub-agent if it's still running — just note it

### 6. Collect Raw Results

As each sub-agent completes, capture its output (the `processing_report` structure).

**Store in dispatch_results:**

```yaml
dispatch_results:
  - story_id: "{story-id-1}"
    batch: 1
    status: "completed"
    raw_output: "{full sub-agent output}"
    report: {parsed processing_report}
  - story_id: "{story-id-2}"
    batch: 1
    status: "failed"
    raw_output: "{full sub-agent output}"
    report: {parsed processing_report or null}
```

### 7. Quick Status to User

Show progress during/after batch execution:

```
🔧 DOCUMENT PROCESSOR DISPATCH STATUS
═══════════════════════════════════════

Batch 1 of {total_batches}:
  {story-id-1}: ✅ Completed — {artifacts_count} artifacts
  {story-id-2}: ✅ Completed — {artifacts_count} artifacts
  {story-id-3}: 🔄 Running...

Batch 2 of {total_batches}: ⏳ Waiting for batch 1

Active: {running_count}/{max_parallel_agents}
Completed: {completed_count}/{total_count}
```

### 8. Proceed to Output Collection

Once ALL batches have finished (completed, failed, or timed out):

```
✅ All document processors finished.
   Completed: {count}, Failed: {count}, Timeout: {count}
   Loading Step 4: Output Collection...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ All stories dispatched within concurrency limits
- ✅ Each sub-agent received complete contract with correct output path
- ✅ Batch dependency ordering respected
- ✅ All sub-agent outputs captured
- ✅ Dispatch status tracked for every story

## FAILURE MODES

- ❌ Dispatching more sub-agents than `{max_parallel_agents}`
- ❌ Not including the full contract in the Task prompt
- ❌ Not capturing sub-agent output for failed/timed-out agents
- ❌ Not resolving placeholders in the contract before dispatch
- ❌ Dispatching dependent stories before their dependencies complete
- ❌ Sub-agents performing git operations or modifying original files
