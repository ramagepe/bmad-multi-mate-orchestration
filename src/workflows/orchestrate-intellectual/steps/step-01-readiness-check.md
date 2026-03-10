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
- 🛑 ALWAYS execute steps in EXACT order: 0 → 1 → 2 → ... Do NOT skip step 0 (pre-flight)
- 📖 Read EVERY story file completely before making readiness judgments
- 🚫 NEVER auto-proceed past failures — always present findings to user
- 🎯 This step runs SEQUENTIALLY — orchestrator only, no sub-agents

---

## EXECUTION SEQUENCE

### 0. Pre-flight: BMM Artifact Checks

🛑 **MANDATORY — Execute this step FIRST before any input validation.**

Before proceeding with standard validation, check for prerequisite BMM artifacts that should exist before orchestration begins. These checks are **advisory only** — they inform but do NOT block.

#### 0a. Implementation Readiness Report

```
Search: {output_folder}/planning-artifacts/implementation-readiness-report-*.md

IF found:
  Read report → extract "Overall Readiness Status" + "Critical Issues"
  Display inline:
    IF READY:  "✅ Advisory: Implementation readiness check completed: READY"
    IF NEEDS WORK: "⚠️ Advisory: Readiness check found issues: [list critical issues].
                    Report: {path}. Recommend addressing before proceeding."

IF not found:
  Display: "⚠️ Advisory: No implementation readiness report found.
           Recommendation: Run 'check-implementation-readiness' workflow first."
```

#### 0b. Sprint Status

```
Search: {output_folder}/implementation-artifacts/sprint-status.yaml

IF found:
  Read file → extract project name, story count, and status summary
  Display inline:
    "✅ Advisory: Sprint status found. {story_count} stories tracked. 
     Stories ready-for-dev: {count}. Stories in backlog: {count}."

IF not found:
  Display: "⚠️ Advisory: No sprint-status.yaml found.
           Recommendation: Run 'sprint-planning' workflow (SM agent) first.
           Sprint planning generates the tracking file for story status."
```

#### 0d. Architecture and PRD Discovery

```
Search: {output_folder}/planning-artifacts/architecture.md
  IF found → store as {architecture_path}
  IF not found → search: {output_folder}/planning-artifacts/architecture*/*.md (sharded)
  IF not found → {architecture_path} = null (advisory: creation panel handles gracefully)

Search: {output_folder}/planning-artifacts/prd.md
  IF found → store as {prd_path}
  IF not found → search: {output_folder}/planning-artifacts/*prd*.md
  IF not found → {prd_path} = null (advisory: creation panel handles gracefully)

Display:
  "Architecture: {✅ FOUND path / ⚠️ NOT FOUND}"
  "PRD:          {✅ FOUND path / ⚠️ NOT FOUND}"
```

#### 0e. Architecture Quality Advisory

🛑 **Advisory only — NEVER blocks the workflow.** This scan surfaces upstream quality signals that may cause sub-agents to improvise or produce inconsistent output. The user can choose to fix the architecture doc before proceeding or accept the risk.

**Skip entirely if `{architecture_path}` is null** (nothing to scan).

IF `{architecture_path}` was found, read the file content (already loaded in 0d) and run these 5 checks:

**P1 — Unresolved versions:** Search for the word `latest` used as a version specifier in the context of dependencies, packages, or technical stack sections. Examples: `"drizzle-orm": "latest"`, `version: latest`, `use the latest version`. Each match is a warning — sub-agents may invent concrete version numbers from training data.

**P2 — Explicit placeholders:** Search for `TBD`, `TODO`, `FIXME`, `PLACEHOLDER`, `[pending]`, or `...` (three dots as standalone content, not in code blocks or ellipsis in prose). Each match is a warning — these represent decisions not yet taken that sub-agents will resolve on their own.

**P3 — Empty sections:** Identify any markdown header (`##`, `###`, `####`) that is immediately followed by another header of the same or higher level with no meaningful content between them (whitespace-only or blank lines don't count as content). Each match is a warning — sub-agents will improvise in that area without guidance.

**P4 — Key area coverage:** The story creation panel (Phase A3) extracts context from these areas: technical stack, code structure, API patterns, database schemas, security, performance, testing, deployment, integration. For each area, check that at least one header in the architecture doc contains a matching keyword:

| Area | Keywords to match in headers |
|------|------------------------------|
| Technical stack | `stack`, `tech`, `dependencies`, `libraries` |
| Code structure | `structure`, `folder`, `directory`, `layout`, `organization` |
| API patterns | `api`, `endpoint`, `route`, `service` |
| Database | `database`, `schema`, `model`, `data` |
| Security | `security`, `auth`, `authorization` |
| Testing | `test`, `quality`, `coverage` |
| Deployment | `deploy`, `ci`, `cd`, `pipeline`, `infrastructure` |

Missing areas are warnings — sub-agents will have no architectural guidance for those domains. Not every project needs every area (a CLI has no API section), so present as advisory, not error.

**P5 — Structure section without concrete paths:** IF a structure/folder section exists (matched in P4), check that it contains at least one concrete file path (a string containing `/` in a non-URL context, e.g., `src/`, `packages/api/`, or a directory tree using `├──`/`└──`). If the section exists but has no concrete paths, warn — sub-agents need literal paths to generate accurate File Lists.

**Display results inline:**

```
IF any warnings found:
  Architecture Quality Advisory:
    {For each warning:}
    ⚠️ [{P1-P5}] {description} (line {line_number_or_range})

  ℹ️ These are advisory only — stories created from this architecture
     may inherit these gaps. Consider fixing before proceeding.

IF no warnings:
  Architecture Quality Advisory: ✅ No quality signals detected.
```

Store warnings in `{architecture_quality_warnings}` for reference in step-08 summary report.

#### 0c. Pre-flight Summary

```
Display combined summary:

  📋 PRE-FLIGHT CHECKS
  ═══════════════════════════════════
  Implementation Readiness Report: {✅ FOUND / ⚠️ NOT FOUND}
  Sprint Status:                   {✅ FOUND / ⚠️ NOT FOUND}
  Architecture:                    {✅ FOUND / ⚠️ NOT FOUND}
  PRD:                             {✅ FOUND / ⚠️ NOT FOUND}

  {IF architecture_quality_warnings is not empty:}
  Architecture Quality Advisory ({warning_count} signals):
    {For each warning:}
    ⚠️ [{check_id}] {description}

  {/IF}

  {IF any BMM artifacts not found:}
  These are advisory only — you may proceed without them.
  For the standard BMAD flow, run these BMM workflows first:
    1. check-implementation-readiness (SM or PM agent)
    2. sprint-planning (SM agent)

DO NOT BLOCK — proceed to step 1
```

### 1. Resolve and Validate Inputs

The orchestrator MUST attempt to auto-discover inputs before asking the user. Only ask the user for information that cannot be resolved automatically.

#### 1a. Resolve epic_path

```
Auto-discovery sequence:
1. Check if user's trigger message mentions a specific epic name/number
2. Search: {output_folder}/planning-artifacts/epics.md
3. IF found → use it as {epic_path}
4. IF NOT found → search: {output_folder}/planning-artifacts/epic*.md
5. IF multiple found → present list to user and ask which one
6. IF none found → ask user for the path
```

#### 1b. Resolve stories

```
Auto-discovery sequence:
1. Read {epic_path} and identify the epic the user requested
   (e.g., "Epic 1" → stories with prefix "1-")
2. Extract ALL story identifiers from the epic section in {epic_path}
   (e.g., "Story 1.1: Monorepo Setup" → story_key "1-1-monorepo-setup")
3. For EACH story identifier, check if a story FILE already exists:
   Search: {output_folder}/implementation-artifacts/stories/{story_key}.md
4. Classify each story:
   - story_file_exists: true  → path to the existing file (ready for REFINEMENT)
   - story_file_exists: false → story exists only as outline in epic (needs CREATION)
5. Present discovery results:

   📋 STORY DISCOVERY — Epic {epic_num}
   ═══════════════════════════════════
   Stories found in epic: {count}

   With existing story files (refinement candidates):
     📄 {story_key}: {story_file_path}
   
   Outline only — no story file (creation candidates):
     📝 {story_key}: exists in epics.md only

6. Use ALL discovered stories as {stories} list
7. Present to user for confirmation before proceeding
```

#### 1c. Resolve remaining inputs

```yaml
required:
  epic_path: "{epic_path}"            # Resolved in 1a
  stories: "{stories}"                # Resolved in 1b
  parallel_limit: "{parallel_limit}"  # From config: max_parallel_agents (auto)
```

**Validation checks:**
- Confirm `{epic_path}` file exists — read it completely
- Confirm each path in `{stories}` list exists — read each completely
- Load `{parallel_limit}` from bmo config (`max_parallel_agents`)
- Confirm `{output_folder}` path from config — create directory if it doesn't exist

**IF any input is missing or invalid after auto-discovery:**
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

**For stories WITH existing files** (`story_file_exists: true`), read the complete file and check:

| Criterion | What to Check | Required? |
|-----------|---------------|-----------|
| **Acceptance Criteria** | Story has explicit, testable ACs | YES |
| **Clarity** | Requirements are unambiguous, no "TBD" in critical fields | YES |
| **Scope Bounded** | Clear boundaries on what's in/out of scope | YES |
| **Dependencies Listed** | Cross-story or external dependencies identified | YES |
| **Output Format Known** | What artifact type is expected (story, spec, analysis) | RECOMMENDED |
| **Terminology Defined** | Key terms are defined or reference a glossary | RECOMMENDED |

**Readiness classification:**
- **READY** — All required criteria met, recommended criteria mostly met
- **PARTIAL** — Required criteria met but with gaps in recommended
- **NOT_READY** — Missing one or more required criteria

**For stories WITHOUT files — outline only** (`story_file_exists: false`), read the story section from the epic file and check:

| Criterion | What to Check | Required? |
|-----------|---------------|-----------|
| **User Story** | Has As a / I want / So that (or equivalent) | YES |
| **Given/When/Then** | Has at least one AC in BDD format | YES |
| **Dependencies** | Lists dependencies on other stories | RECOMMENDED |
| **Labels** | Has category labels | RECOMMENDED |

**Readiness classification:**
- **OUTLINE_READY** — User story + at least one BDD AC present → proceeds with `story_creation`
- **OUTLINE_INCOMPLETE** — Missing user story or all ACs → flagged for user decision

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

**Auto-proceed condition:** IF ALL stories are READY or OUTLINE_READY (zero PARTIAL, zero NOT_READY, zero OUTLINE_INCOMPLETE) AND no critical cross-story concerns were flagged → skip the interactive gate. Display:

```
✅ All {count} stories {READY/OUTLINE_READY} — auto-proceeding (no exclusions needed).
```

Then go directly to step 7 (Store Readiness State) with all stories included.

**Interactive gate — ONLY if there are stories that are PARTIAL, NOT_READY, or OUTLINE_INCOMPLETE, OR if critical cross-story concerns exist:**

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
  stories_proceeding:
    - story_id: "{story-key}"
      story_file_exists: true          # true = refinement candidate, false = creation candidate
      story_file_path: "{path}"        # Path to existing file (if story_file_exists: true)
      readiness: "READY"               # READY | PARTIAL | NOT_READY | OUTLINE_READY | OUTLINE_INCOMPLETE
  stories_excluded: list[{story_id, reason}]
  cross_story_concerns: list[string]   # Flagged for cross-validation (step 5)
  epic_context: string                 # Summary of epic for sub-agents
  epic_path: "{epic_path}"             # Carried forward for creation contracts
  architecture_path: "{path}"          # Discovered in 0d, used in step-02/03 contracts
  prd_path: "{path}"                   # Discovered in 0d, used in step-02/03 contracts
  output_folder: "{output_folder}"     # From config
  stories_output_path: "{output_folder}/implementation-artifacts/stories"  # Where story files go
  architecture_quality_warnings: list[{check_id, description, line}]  # From 0e advisory scan (may be empty)
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
