---
name: step-01-readiness-check
description: "Validate stories are implementation-ready before fan-out"
nextStepFile: './step-02-shared-file-snapshot.md'
phase: sequential
phase_number: 1
executor: orchestrator
---

# Step 1: Readiness Check

**Progress: Step 1 of 14** — Next: Shared File Snapshot
**Phase:** 1 (SEQUENTIAL — Orchestrator Only)

---

## STEP GOAL

Validate that all stories in `{stories}` are implementation-ready before committing resources to worktree creation and sub-agent dispatch. This is the first quality gate — it prevents wasted work on incomplete specs.

---

## MANDATORY EXECUTION RULES

- 🛑 NEVER skip readiness validation — incomplete stories cause cascading failures
- 📖 Read EVERY story file completely before making readiness judgments
- 🚫 NEVER auto-proceed past failures — always present findings to user
- 🎯 This step runs SEQUENTIALLY — orchestrator only, no sub-agents

---

## EXECUTION SEQUENCE

### 1. Validate Input Contract

Verify all required inputs are available:

```yaml
required:
  epic_path: "{epic_path}"        # Must exist and be readable
  stories: "{stories}"            # Must be non-empty list
  base_branch: "{base_branch}"    # Must be a valid git branch
  parallel_limit: "{parallel_limit}"  # From config: max_parallel_agents
```

**Validation checks:**
- Confirm `{epic_path}` file exists — read it completely
- Confirm each path in `{stories}` list exists — read each completely
- Confirm `{base_branch}` exists: run `git branch --list {base_branch}`
- Confirm we are NOT currently on `{base_branch}` (safety check)
- Load `{parallel_limit}` from bmo config (`max_parallel_agents`)

**IF any input is missing or invalid:**
- Report the specific missing/invalid inputs to user
- STOP — do not proceed until resolved

### 2. Read Epic File

Load and read `{epic_path}` completely. Extract:
- Epic goal / description
- List of stories referenced in the epic
- Any shared architecture decisions or constraints
- Integration points between stories

Store this context — it will be passed to sub-agents and cross-validator.

### 3. Validate Each Story for Implementation Readiness

For EACH story in `{stories}`, read the complete file and check:

| Criterion | What to Check | Required? |
|-----------|---------------|-----------|
| **Acceptance Criteria** | Story has explicit, testable ACs | YES |
| **Architecture Defined** | Technical approach is specified (not just "TBD") | YES |
| **Scope Bounded** | Clear boundaries on what's in/out of scope | YES |
| **Dependencies Listed** | Any cross-story or external dependencies identified | YES |
| **File Scope Identified** | Which files/modules will be touched | RECOMMENDED |
| **Test Strategy** | How to verify the implementation | YES |

**Readiness classification per story:**
- **READY** — All required criteria met, recommended criteria mostly met
- **PARTIAL** — Required criteria met but with gaps in recommended
- **NOT_READY** — Missing one or more required criteria

### 4. Identify Cross-Story Concerns

Analyze all stories together for:
- **Shared file overlap**: Multiple stories touching the same files (flag for step 9)
- **Dependency chains**: Story A depends on Story B's output
- **Scope conflicts**: Contradictory requirements between stories
- **Parallel safety**: Can these stories safely run in parallel?

### 5. Present Readiness Report to User

Display a clear summary:

```
📋 READINESS REPORT — {epic_path}
═══════════════════════════════════════

Stories analyzed: {count}
Ready: {ready_count}
Partial: {partial_count}
Not Ready: {not_ready_count}

┌──────────────────┬──────────┬─────────────────────┐
│ Story            │ Status   │ Issues              │
├──────────────────┼──────────┼─────────────────────┤
│ {story-id-1}     │ ✅ READY │ —                   │
│ {story-id-2}     │ ⚠️ PARTIAL│ Missing test strategy│
│ {story-id-3}     │ ❌ NOT   │ No ACs defined      │
└──────────────────┴──────────┴─────────────────────┘

⚠️ Cross-Story Concerns:
- {any shared file overlaps}
- {any dependency chains}
- {any scope conflicts}

Recommendation: Proceed with {ready_count + partial_count} stories,
exclude {not_ready_count} until resolved.
```

### 6. User Decision Gate

Present options:

```
What would you like to do?

[A] Proceed with ALL stories (including partial/not-ready — at your risk)
[R] Proceed with READY + PARTIAL stories only (recommended)
[S] Select specific stories to include
[X] Abort — fix stories first
```

**Handle response:**
- **A**: Warn user about risks, confirm, then proceed with all
- **R**: Filter to ready + partial stories, update `{stories}` list
- **S**: Let user pick specific stories, update `{stories}` list
- **X**: STOP workflow — report what needs fixing

### 7. Store Readiness State

After user decision, store the execution state:

```yaml
readiness_state:
  stories_proceeding: list[string]   # Final list of stories moving forward
  stories_excluded: list[string]     # Stories excluded and why
  cross_story_concerns: list[string] # Flagged for later steps
  shared_file_candidates: list[string] # Files touched by multiple stories
  epic_context: string               # Summary of epic for sub-agents
```

### 8. Proceed to Next Step

Confirm to user:
```
✅ Readiness check complete.
   Proceeding with {count} stories: {story_ids}
   Loading Step 2: Shared File Snapshot...
```

Load, read completely, then execute `{nextStepFile}`.

---

## QUALITY GATE: READINESS CHECK

| Validates | Failure Action |
|-----------|----------------|
| Stories have complete ACs | Flag incomplete, proceed with ready |
| Architecture is defined | Flag undefined, proceed with ready |
| No critical cross-story conflicts | Alert user, let them decide |

---

## SUCCESS METRICS

- ✅ Every story file was read completely
- ✅ Each story classified as READY/PARTIAL/NOT_READY
- ✅ Cross-story concerns identified
- ✅ User made informed decision on which stories to proceed with
- ✅ Execution state stored for downstream steps

## FAILURE MODES

- ❌ Skipping story file reading
- ❌ Auto-proceeding without user decision on exclusions
- ❌ Not identifying shared file overlaps (causes issues in step 9)
- ❌ Proceeding with stories that have no acceptance criteria
