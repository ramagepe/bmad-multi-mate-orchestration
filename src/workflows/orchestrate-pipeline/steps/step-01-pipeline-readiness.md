---
name: step-01-pipeline-readiness
description: "Validate epic, stories, and sub-workflows exist; collect inputs for both phases"
nextStepFile: './step-02-intellectual-phase.md'
phase: sequential
phase_number: 1
executor: orchestrator
---

# Step 1: Pipeline Readiness Check

**Progress: Step 1 of 5** — Next: Intellectual Phase
**Phase:** 1 (SEQUENTIAL — Orchestrator Only)

---

## STEP GOAL

Validate that the pipeline can run end-to-end before committing to either phase. This means confirming the epic and stories exist, both sub-workflows are available and implemented, and collecting all inputs needed for BOTH the intellectual phase (epic_path, stories, parallel_limit) and the implementation phase (base_branch). This is a lighter readiness check than each sub-workflow performs — the sub-workflows will do their own detailed readiness checks internally.

---

## MANDATORY EXECUTION RULES

- 🛑 NEVER skip sub-workflow existence validation — a missing sub-workflow means a broken pipeline
- 📖 Read the epic file and verify story files exist — but leave deep story validation to the sub-workflows
- 🚫 NEVER auto-proceed if either sub-workflow is missing or not implemented
- 🎯 Collect ALL inputs upfront — the user should not be interrupted mid-pipeline for missing config
- ⚠️ base_branch MUST be collected here — it's needed for the implementation phase (step 4)

---

## EXECUTION SEQUENCE

### 1. Validate Input Contract

Verify all required inputs are available or can be collected:

```yaml
required:
  epic_path: "{epic_path}"        # Must exist and be readable
  stories: "{stories}"            # Must be non-empty list
  parallel_limit: "{parallel_limit}"  # From config: max_parallel_agents
  base_branch: "{base_branch}"    # Needed for implementation phase
```

**Validation checks:**
- Confirm `{epic_path}` file exists — read it to verify it has meaningful content
- Confirm each path in `{stories}` list exists and is readable
- Load `{parallel_limit}` from bmo config (`max_parallel_agents`)
- Load `{output_folder}` from bmo config — create directory if it doesn't exist

**IF any input is missing or invalid:**
- Report the specific missing/invalid inputs to user
- STOP — do not proceed until resolved

### 2. Read and Validate Epic File

Load and read `{epic_path}` completely. Extract:
- Epic goal / description
- List of stories referenced in the epic
- Shared constraints, terminology, or definitions
- Dependencies between stories

**Validate epic content:**
- IF the epic file is empty or contains no meaningful content:
  - Report: "Epic file `{epic_path}` exists but has no meaningful content."
  - STOP — an empty epic provides no context for either phase.
  - Ask user to provide a valid epic file before continuing.

Store epic context — it will be passed through to both sub-workflows.

### 3. Validate Story Files Exist

For EACH story in `{stories}`:
- Confirm the file exists and is readable
- Read the file to confirm it has content (not empty)
- Do NOT perform detailed readiness classification here — that's the sub-workflow's job

**IF any story file is missing or empty:**
- Report which story files are missing/empty
- Ask user to fix or remove them from the list before continuing

### 4. Validate Sub-Workflows Exist and Are Implemented

Check that both sub-workflows are available:

```yaml
sub_workflows:
  intellectual:
    path: "{project-root}/_bmad/bmo/workflows/orchestrate-intellectual/workflow.md"
    required_status: implemented
    first_step: "./steps/step-01-readiness-check.md"
  implementation:
    path: "{project-root}/_bmad/bmo/workflows/orchestrate-implementation/workflow.md"
    required_status: implemented
    first_step: "./steps/step-01-readiness-check.md"
```

**For each sub-workflow:**
- Read `workflow.md` and confirm `status: implemented` in frontmatter
- Confirm `firstStep` path exists and is readable
- Count step files in `./steps/` directory to verify completeness

**IF either sub-workflow is missing or not implemented:**
- Report: "Sub-workflow `{name}` at `{path}` is {missing / status: spec}. Cannot run pipeline."
- STOP — both sub-workflows must be implemented for the pipeline to function.

### 5. Collect base_branch from User

The implementation phase requires a `base_branch`. Collect it now:

```
🔧 PIPELINE CONFIGURATION
═══════════════════════════════════════

The full pipeline will run two phases:
  Phase 1: Intellectual — document refinement, cross-validation
  Phase 2: Implementation — worktree-isolated coding, review, merge

Implementation phase requires a base branch for worktree creation.

Current branch: {result of git branch --show-current}

What base branch should be used for implementation worktrees?
  Default: {current_branch}
  
Enter branch name or press Enter for default:
```

**Validate the branch:**
- Confirm branch exists: `git rev-parse --verify {base_branch}`
- IF branch doesn't exist: report error, ask user to provide a valid branch

### 6. Story Count Hard Limit

**IF `{story_count}` > 4 — HARD STOP:**

The pipeline is NOT available for more than 4 stories. This is a technical limitation of v1: the full pipeline runs 27 step files + sub-agent dispatches + correction loops + user gates in a single context window. Beyond 4 stories, context pressure degrades output quality in later steps to an unacceptable degree.

```
🛑 STORY COUNT LIMIT EXCEEDED
═══════════════════════════════════════

You have {story_count} stories — the pipeline supports a maximum of 4.

The full pipeline (27 step files + sub-agent dispatches + corrections +
user gates) cannot maintain output quality beyond 4 stories in a single
context window. This is a hard limit in v1, not a suggestion.

To proceed with {story_count} stories:
  • Run [OI] (intellectual phase) independently
  • Run [OD] (implementation phase) independently
  • Split stories into batches of 3-4 if needed

Pipeline mode is not available for this story count.
═══════════════════════════════════════
```

**STOP** — do not allow the user to override this limit. Advise them to use [OI] and [OD] from the orchestrator menu instead.

### 7. Confirm Pipeline Mode

Present the pipeline configuration for user confirmation:

```
📋 PIPELINE READINESS SUMMARY
═══════════════════════════════════════

Epic: {epic_path}
Stories: {count} stories
Mode: Mixed (intellectual → implementation)
Base Branch: {base_branch}
Parallel Limit: {parallel_limit}
Output Folder: {output_folder}

Sub-Workflows:
  ✅ orchestrate-intellectual — implemented ({step_count} steps)
  ✅ orchestrate-implementation — implemented ({step_count} steps)

Pipeline Steps:
  1. ✅ Pipeline Readiness (this step)
  2. 🔜 Intellectual Phase (document refinement)
  3. 🔜 Phase Gate (approval to proceed)
  4. 🔜 Implementation Phase (code in worktrees)
  5. 🔜 Final Summary (combined report)

═══════════════════════════════════════

⚠️ The full pipeline may take significant time depending on story count.

Ready to begin?

[P] Proceed with full pipeline
[I] Run intellectual phase only (use [OI] menu instead)
[X] Abort — not ready
```

**Handle response:**
- **P**: Proceed with full pipeline
- **I**: Advise user to use `[OI]` from the orchestrator menu instead. STOP pipeline.
- **X**: STOP — report what was validated and user chose to abort.

### 8. Store Pipeline State

After user confirms, store the execution state:

```yaml
pipeline_state:
  mode: mixed
  epic_path: "{epic_path}"
  epic_context: "{epic summary}"
  stories: ["{story paths}"]
  story_count: int
  parallel_limit: int
  base_branch: "{base_branch}"
  output_folder: "{output_folder}"
  intellectual_workflow_path: "{path}"
  implementation_workflow_path: "{path}"
  pipeline_start_timestamp: "{timestamp}"
  phase: "starting_intellectual"
  intellectual_rerun_count: 0         # Tracks re-runs from phase gate (max 2)
```

### 9. Proceed to Next Step

Confirm to user:
```
✅ Pipeline readiness check complete.
   {story_count} stories validated.
   Both sub-workflows confirmed.
   
   Starting Phase 1: Intellectual...
   Loading Step 2: Intellectual Phase...
```

Load, read completely, then execute `{nextStepFile}`.

---

## QUALITY GATE: PIPELINE READINESS

| Validates | Failure Action |
|-----------|----------------|
| Epic file exists and has content | STOP — user must provide valid epic |
| All story files exist | STOP — user must fix paths |
| Both sub-workflows implemented | STOP — cannot run incomplete pipeline |
| Base branch exists | STOP — user must provide valid branch |
| User confirms pipeline mode | STOP on abort |

---

## SUCCESS METRICS

- ✅ Epic file read and validated
- ✅ All story files confirmed to exist
- ✅ Both sub-workflows confirmed as implemented
- ✅ base_branch collected and validated
- ✅ User confirmed pipeline mode
- ✅ Pipeline state stored for downstream steps

## FAILURE MODES

- ❌ Not validating sub-workflow existence (pipeline breaks at phase transition)
- ❌ Not collecting base_branch (implementation phase cannot start)
- ❌ Performing deep story readiness checks (that's the sub-workflow's job)
- ❌ Auto-proceeding without user confirmation
- ❌ Not reading the epic file
