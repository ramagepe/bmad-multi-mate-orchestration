---
name: step-02-fan-out-planning
description: "Plan sub-agent assignments, workflow selection, and parallel batching"
nextStepFile: './step-03-sub-agent-dispatch.md'
phase: sequential
phase_number: 1
executor: orchestrator
---

# Step 2: Fan-Out Planning

**Progress: Step 2 of 8** — Next: Sub-Agent Dispatch
**Phase:** 1 (SEQUENTIAL — Orchestrator Only)

---

## STEP GOAL

Determine which BMAD workflow each sub-agent will invoke, assign output paths, resolve dependency ordering, and group stories into parallel batches respecting the concurrency limit. This step produces the complete dispatch plan for step 3.

---

## MANDATORY EXECUTION RULES

- 🛑 NEVER dispatch sub-agents in this step — planning only
- 📖 Consider dependency chains when forming batches
- 🚫 NEVER assign more sub-agents per batch than `{max_parallel_agents}`
- 🎯 Each story gets exactly one sub-agent assignment with a specific BMAD workflow

---

## EXECUTION SEQUENCE

### 1. Determine Task Type and Workflow per Story

For EACH story in `{stories_proceeding}`, analyze the story content and determine:

| Story Characteristic | Task Type | Workflow to Invoke |
|---------------------|-----------|--------------------|
| Story needs full creation from an outline | `story_refinement` | `create-story` |
| Story needs deep requirements elicitation | `analysis` | `advanced-elicitation` |
| Story needs document creation (spec, ADR, etc.) | `document_creation` | Context-dependent |
| Story is already well-defined, needs polishing | `story_refinement` | `create-story` |

**For each story, record:**

```yaml
assignment:
  story_id: "{story-id}"
  story_file_path: "{path}"
  task_type: "story_refinement"  # or "document_creation" or "analysis"
  workflow_to_invoke: "create-story"  # specific BMAD workflow
  output_path: "{output_folder}/{story-id}/"
  dependencies: list[string]  # story IDs this story depends on
  estimated_complexity: "low"  # low | medium | high
```

### 2. Create Output Directory Structure

Plan the output directory structure:

```
{output_folder}/
├── {story-id-1}/
│   ├── {artifact files will go here}
│   └── _processing_log.md
├── {story-id-2}/
│   └── ...
└── _orchestration/
    └── cross-validation-report.md
```

**Create the directory structure:**
- Create `{output_folder}` if it doesn't exist
- Create a subdirectory for each story: `{output_folder}/{story-id}/`
- Create `{output_folder}/_orchestration/` for cross-validation artifacts

### 3. Resolve Dependency Ordering

Analyze the dependency graph from `{cross_story_concerns}` and story analysis:

```
IF Story B depends on Story A's output:
  → Story A must complete BEFORE Story B can start
  → Place Story A in an earlier batch than Story B

IF no dependencies exist between stories:
  → All stories can run in the same batch (up to concurrency limit)
```

**Build the dependency-ordered queue:**

```yaml
dependency_order:
  independent: list[string]     # Stories with no dependencies — can run first
  dependent: list[{story_id, depends_on: list[string]}]  # Must wait
```

### 4. Form Parallel Batches

Group stories into batches respecting:
1. **Concurrency limit**: No batch exceeds `{max_parallel_agents}` stories
2. **Dependencies**: Dependent stories run in later batches than their dependencies
3. **Complexity balancing**: Mix high/low complexity within a batch when possible

```yaml
dispatch_plan:
  batch_count: int
  batches:
    - batch_number: 1
      stories:
        - story_id: "{story-id-1}"
          assignment: {from step 1}
        - story_id: "{story-id-2}"
          assignment: {from step 1}
        - story_id: "{story-id-3}"
          assignment: {from step 1}
    - batch_number: 2
      stories:
        - story_id: "{story-id-4}"  # Depends on story-id-1
          assignment: {from step 1}
      wait_for: ["{story-id-1}"]    # Must wait for these to complete
```

**Edge case — all stories independent, count ≤ max_parallel_agents:**
→ Single batch with all stories

**Edge case — all stories independent, count > max_parallel_agents:**
→ Split into batches of `{max_parallel_agents}` each

**Edge case — deep dependency chain (A → B → C):**
→ Three sequential batches, one story each

### 5. Present Fan-Out Plan to User

```
📐 FAN-OUT PLAN — {stories_proceeding.count} stories
═══════════════════════════════════════

Max parallel agents: {max_parallel_agents}
Total batches: {batch_count}
Output folder: {output_folder}

Batch 1 ({batch_1_count} stories — parallel):
  📄 {story-id-1} → {workflow_to_invoke} → {output_path}
  📄 {story-id-2} → {workflow_to_invoke} → {output_path}
  📄 {story-id-3} → {workflow_to_invoke} → {output_path}

Batch 2 ({batch_2_count} stories — waits for batch 1):
  📄 {story-id-4} → {workflow_to_invoke} → {output_path}
  ⏳ Depends on: {story-id-1}

{if dependency_notes}
📌 Dependency Notes:
- {dependency_explanation}
{/if}

Proceed with this plan? [Y/N/E]
  Y = Yes, start dispatch
  N = No, abort
  E = Edit — reassign workflows or reorder
```

### 6. Handle User Response

- **Y**: Confirm plan and proceed to dispatch
- **N**: Abort workflow
- **E**: Let user modify assignments:
  - Change workflow for a specific story
  - Reorder batches
  - Remove a story from the plan
  - After edits, re-present the updated plan for confirmation

### 7. Store Fan-Out Plan

```yaml
fanout_state:
  dispatch_plan: {complete plan from step 4}
  assignments: list[{story_id, task_type, workflow_to_invoke, output_path}]
  total_batches: int
  estimated_total_time: string  # Rough estimate based on batch count × timeout
  user_confirmed: true
  # Carried forward from step 1 (must be available for step 3 dispatch contracts):
  epic_context: "{readiness_state.epic_context}"
  cross_story_concerns: "{readiness_state.cross_story_concerns}"
```

### 8. Proceed to Next Step

```
✅ Fan-out plan confirmed.
   {batch_count} batches, {stories_count} stories total.
   Loading Step 3: Sub-Agent Dispatch...
```

Load, read completely, then execute `{nextStepFile}`.

---

## SUCCESS METRICS

- ✅ Every story has a clear workflow assignment
- ✅ Output paths created for all stories
- ✅ Dependencies resolved and batches ordered correctly
- ✅ No batch exceeds concurrency limit
- ✅ User confirmed the plan before dispatch

## FAILURE MODES

- ❌ Assigning a workflow that doesn't exist in the project
- ❌ Creating batches that exceed `{max_parallel_agents}`
- ❌ Ignoring dependency chains (dispatching B before A completes)
- ❌ Not creating output directories before dispatch
- ❌ Proceeding without user confirmation of the plan
