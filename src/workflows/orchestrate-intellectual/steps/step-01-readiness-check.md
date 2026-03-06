---
name: step-01-readiness-check
description: "Validate stories are ready for document processing before fan-out"
nextStepFile: './step-02-fan-out-planning.md'
phase: sequential
phase_number: 1
executor: orchestrator
---

# Step 1: Readiness Check

**Progress: Step 1 of 8** — Next: Fan-Out Planning
**Phase:** 1 (SEQUENTIAL — Orchestrator Only)

---

## STEP GOAL

Validate that all stories in `{stories}` are ready for document processing before committing resources to sub-agent dispatch. This is the first quality gate — it prevents wasted work on incomplete or ambiguous specs.

---

## MANDATORY EXECUTION RULES

- 🛑 NEVER skip readiness validation — incomplete stories cause cascading failures in document processing
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
  parallel_limit: "{parallel_limit}"  # From config: max_parallel_agents
```

**Validation checks:**
- Confirm `{epic_path}` file exists — read it completely
- Confirm each path in `{stories}` list exists — read each completely
- Load `{parallel_limit}` from bmo config (`max_parallel_agents`)
- Confirm `{output_folder}` path from config — create directory if it doesn't exist

**IF any input is missing or invalid:**
- Report the specific missing/invalid inputs to user
- STOP — do not proceed until resolved

### 2. Read Epic File

Load and read `{epic_path}` completely. Extract:
- Epic goal / description
- List of stories referenced in the epic
- Any shared constraints, terminology, or definitions
- Dependencies between stories
- Expected deliverables / artifacts

**Validate epic content:**
- IF the epic file is empty or contains no meaningful content (no goals, no story references):
  - Report: "Epic file `{epic_path}` exists but has no meaningful content."
  - STOP — an empty epic provides no context for sub-agents or cross-validation.
  - Ask user to provide a valid epic file before continuing.

Store this context — it will be passed to sub-agents and cross-validator.

### 3. Validate Each Story for Document Processing Readiness

For EACH story in `{stories}`, read the complete file and check:

| Criterion | What to Check | Required? |
|-----------|---------------|-----------|
| **Acceptance Criteria** | Story has explicit, testable ACs | YES |
| **Clarity** | Requirements are unambiguous, no "TBD" in critical fields | YES |
| **Scope Bounded** | Clear boundaries on what's in/out of scope | YES |
| **Dependencies Listed** | Cross-story or external dependencies identified | YES |
| **Output Format Known** | What artifact type is expected (story, spec, analysis) | RECOMMENDED |
| **Terminology Defined** | Key terms are defined or reference a glossary | RECOMMENDED |

**Readiness classification per story:**
- **READY** — All required criteria met, recommended criteria mostly met
- **PARTIAL** — Required criteria met but with gaps in recommended
- **NOT_READY** — Missing one or more required criteria

### 4. Identify Cross-Story Concerns

Analyze all stories together for:
- **Terminology conflicts**: Same term used differently across stories
- **Dependency chains**: Story A depends on Story B's output
- **Scope overlaps**: Multiple stories covering the same requirement
- **Contradictions**: Conflicting requirements between stories
- **Coverage gaps**: Epic requirements not covered by any story

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
│ {story-id-2}     │ ⚠️ PARTIAL│ Output format unclear│
│ {story-id-3}     │ ❌ NOT   │ No ACs defined      │
└──────────────────┴──────────┴─────────────────────┘

⚠️ Cross-Story Concerns:
- {any terminology conflicts}
- {any dependency chains}
- {any scope overlaps or contradictions}

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
  cross_story_concerns: list[string] # Flagged for cross-validation (step 5)
  epic_context: string               # Summary of epic for sub-agents
  output_folder: "{output_folder}"   # From config
```

### 8. Proceed to Next Step

Confirm to user:
```
✅ Readiness check complete.
   Proceeding with {count} stories: {story_ids}
   Loading Step 2: Fan-Out Planning...
```

Load, read completely, then execute `{nextStepFile}`.

---

## QUALITY GATE: READINESS CHECK

| Validates | Failure Action |
|-----------|----------------|
| Stories have complete ACs | Flag incomplete, proceed with ready |
| Requirements are unambiguous | Flag ambiguous, proceed with ready |
| No critical cross-story contradictions | Alert user, let them decide |

---

## SUCCESS METRICS

- ✅ Every story file was read completely
- ✅ Each story classified as READY/PARTIAL/NOT_READY
- ✅ Cross-story concerns identified (terminology, overlaps, contradictions)
- ✅ User made informed decision on which stories to proceed with
- ✅ Execution state stored for downstream steps

## FAILURE MODES

- ❌ Skipping story file reading
- ❌ Auto-proceeding without user decision on exclusions
- ❌ Not identifying terminology conflicts or scope overlaps
- ❌ Proceeding with stories that have no acceptance criteria
- ❌ Not creating/validating the output folder
