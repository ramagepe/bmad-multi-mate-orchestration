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

For EACH story in `{stories_proceeding}`, use the `story_file_exists` flag from step 1 readiness state:

**Detection logic:**
1. Check `story_file_exists` from readiness_state for this story
2. IF `story_file_exists == false` → task_type is `story_creation`
3. IF `story_file_exists == true` → analyze content to determine refinement vs other

| Story Characteristic | Task Type | Panel/Workflow |
|---------------------|-----------|--------------------|
| Story has NO file — outline only in epics.md | `story_creation` | `story-creation-panel` |
| Story file exists, needs strengthening | `story_refinement` | `story-refinement-panel` |
| Story needs deep requirements elicitation | `analysis` | `advanced-elicitation` |
| Story needs document creation (spec, ADR, etc.) | `document_creation` | Context-dependent |

**For each story, record:**

```yaml
assignment:
  story_id: "{story-key}"
  task_type: "story_creation"         # or "story_refinement" or "analysis" or "document_creation"
  workflow_to_invoke: "story-creation-panel"  # or "story-refinement-panel" or other
  output_path: "{stories_output_path}" # = {output_folder}/implementation-artifacts/stories
  dependencies: list[string]           # story IDs this story depends on
  estimated_complexity: "low"          # low | medium | high
  # Fields carried from step-01 readiness_state:
  story_file_exists: false             # from readiness_state
  story_file_path: "{path}"           # IF story_file_exists: true
  epic_path: "{epic_path}"            # from readiness_state
  epic_num: "{epic_num}"              # extracted from story_id
  story_num: "{story_num}"            # extracted from story_id
  story_key: "{story_key}"            # e.g., "1-1-monorepo-setup"
  architecture_path: "{path}"         # from readiness_state (may be null)
  prd_path: "{path}"                  # from readiness_state (may be null)
  previous_story_path: "{path or null}" # resolved: previous story in epic sequence — see resolution rules below
```

**`previous_story_path` resolution rules:**
- IF `story_num == 1` (first story in epic) → `null` (no previous)
- IF the previous story (story_num - 1) has `story_file_exists: true` → resolve to its file path
- IF the previous story was assigned to an EARLIER batch (already completed) → resolve to `{stories_output_path}/{previous_story_key}.md`
- IF the previous story is in the SAME batch (running in parallel) → `null` (not available yet)
- IF the previous story is in a LATER batch → `null` (not created yet)

### 2. Verify Output Directory Structure

Stories are written directly to the shared stories folder — NOT to per-story subdirectories. Orchestration artifacts go to a separate `_orchestration/` folder.

```
{stories_output_path}/                    # = {output_folder}/implementation-artifacts/stories/
├── {story_key}.md                        # Each story file (created or refined)
├── {story_key}.md                        # ...one per story
└── ...

{output_folder}/_orchestration/           # BMO orchestration artifacts only
├── cross-validation-report.md
└── orchestration-summary.md
```

**Verify/create the directory structure:**
- Verify `{stories_output_path}` exists (from readiness_state); create if missing
- Create `{output_folder}/_orchestration/` for cross-validation and summary artifacts
- ⚠️ Do NOT create per-story subdirectories — stories are flat files in `{stories_output_path}`

### 3. Resolve Dependency Ordering

Analyze the dependency graph from `{cross_story_concerns}` and story analysis:

> 📘 **INTELLECTUAL MODE CONTEXT:** In intellectual mode, sub-agents process stories as *documents* — whether creating new story files from outlines or refining existing ones. They don't produce code or depend on compiled outputs from other stories. A story that declares a code-level dependency (e.g., "depends on Story 1.1 for monorepo structure") can still be *processed in parallel* with its dependency, because creation/refinement only requires the original outline or story content + epic context, not the processed output.
>
> Apply dependency ordering only when Story B's processing genuinely requires the *output* of Story A (e.g., Story A defines an API contract that Story B's ACs reference and might change during creation/refinement).
>
> **CREATION MODE — MAXIMIZE PARALLELISM:** When all stories have `story_file_exists: false` (creation mode), the authoritative sources are `epics.md` + `architecture.md` + `prd.md` — NOT other story files being created in parallel. The `previous_story_path` field adds marginal value in creation mode because story files don't exist yet and the architecture document already contains all structural decisions. Do NOT split into extra batches just to make `previous_story_path` available — the cross-validator (step 5) handles inter-story coherence. Batch only for `max_parallel_agents` overflow.
>
> In **refinement mode** (existing story files), `previous_story_path` IS valuable — previous stories may contain dev notes, corrections, and established patterns that inform refinement. Sequencing for `previous_story_path` is justified when refining.

```
IF Story B depends on Story A's REFINED OUTPUT (content dependency):
  → Story A must complete BEFORE Story B can start
  → Place Story A in an earlier batch than Story B

IF Story B has only CODE-LEVEL dependencies on Story A (implementation dependency):
  → In intellectual mode, these can be refined in parallel
  → The cross-validator (step 5) will catch any content misalignments

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

**Always display the plan** — the user should see what's about to happen:

```
📐 FAN-OUT PLAN — {stories_proceeding.count} stories
═══════════════════════════════════════

Max parallel agents: {max_parallel_agents}
Total batches: {batch_count}
Output folder: {output_folder}

Batch 1 ({batch_1_count} stories — parallel):
  📄 {story-id-1} → {workflow_to_invoke} → {stories_output_path}/{story_key}.md
  📄 {story-id-2} → {workflow_to_invoke} → {stories_output_path}/{story_key}.md
  📄 {story-id-3} → {workflow_to_invoke} → {stories_output_path}/{story_key}.md

Batch 2 ({batch_2_count} stories — waits for batch 1):
  📄 {story-id-4} → {workflow_to_invoke} → {stories_output_path}/{story_key}.md
  ⏳ Depends on: {story-id-1}

{if dependency_notes}
📌 Dependency Notes:
- {dependency_explanation}
{/if}
```

**Auto-proceed condition:** IF ALL of the following are true → skip the interactive confirmation, display "Auto-proceeding", and go directly to step 7 (Store Fan-Out Plan):
- Single batch (batch_count == 1)
- All stories use the same workflow (no mixed workflow assignments)
- No content-level dependencies required sequencing overrides

```
✅ Straightforward plan — {batch_count} batch, {story_count} stories, all {workflow_name}. Auto-proceeding to dispatch.
```

**Interactive confirmation — ONLY if the plan has complexity that warrants user review:**
- Multiple batches (batching decisions may need adjustment)
- Mixed workflows across stories (user may want to change assignments)
- Content-level dependencies forced specific sequencing (user may disagree)

```
Proceed with this plan? [Y/N/E]
  Y = Yes, start dispatch
  N = No, abort
  E = Edit — reassign workflows or reorder
```

### 6. Handle User Response (only if interactive confirmation was shown)

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
  assignments: list[{story_id, task_type, workflow_to_invoke, output_path, story_key}]
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
- ✅ Plan presented to user (auto-proceeded if straightforward, or user confirmed if complex)

## FAILURE MODES

- ❌ Assigning a workflow that doesn't exist in the project
- ❌ Creating batches that exceed `{max_parallel_agents}`
- ❌ Ignoring dependency chains (dispatching B before A completes)
- ❌ Not creating output directories before dispatch
- ❌ Proceeding without displaying the plan (auto-proceed still shows the plan, just skips the prompt)
