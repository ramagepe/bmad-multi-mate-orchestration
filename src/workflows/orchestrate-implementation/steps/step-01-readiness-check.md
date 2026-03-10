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
  IF not found → {architecture_path} = null (advisory only)

Search: {output_folder}/planning-artifacts/prd.md
  IF found → store as {prd_path}
  IF not found → search: {output_folder}/planning-artifacts/*prd*.md
  IF not found → {prd_path} = null (advisory only)

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

Store warnings in `{architecture_quality_warnings}` for reference in the summary report.

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
2. Search: {output_folder}/implementation-artifacts/stories/{epic_prefix}-*.md
   (e.g., for Epic 1: 1-*.md → 1-1-*.md, 1-2-*.md, etc.)
3. IF stories found → use them as {stories} list
4. IF no stories found → ask user for story paths
5. Present discovered stories to user for confirmation before proceeding
```

#### 1c. Resolve base_branch

```
Auto-discovery sequence:
1. Check if user's trigger message specifies a base branch
2. IF not specified → check if 'dev' branch exists: git branch --list dev
3. IF 'dev' exists → use it as {base_branch}
4. IF 'dev' does not exist → check 'main', then 'master'
5. IF none found → ask user for the base branch
6. Confirm we are NOT currently on {base_branch} (safety check)
```

#### 1d. Resolve remaining inputs

```yaml
required:
  epic_path: "{epic_path}"            # Resolved in 1a
  stories: "{stories}"                # Resolved in 1b
  base_branch: "{base_branch}"       # Resolved in 1c
  parallel_limit: "{parallel_limit}"  # From config: max_parallel_agents (auto)
```

**Validation checks:**
- Confirm `{epic_path}` file exists — read it completely
- Confirm each path in `{stories}` list exists — read each completely
- Confirm `{base_branch}` exists: run `git branch --list {base_branch}`
- Confirm we are NOT currently on `{base_branch}` (safety check)
- Load `{parallel_limit}` from bmo config (`max_parallel_agents`)

**IF any input is missing or invalid after auto-discovery:**
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

**Auto-proceed condition:** IF ALL stories are READY (zero PARTIAL, zero NOT_READY) AND no critical cross-story concerns were flagged → skip the interactive gate. Display:

```
✅ All {count} stories READY — auto-proceeding (no exclusions needed).
```

Then go directly to step 7 (Store Readiness State) with all stories included.

**Interactive gate — ONLY if there are stories that are PARTIAL or NOT_READY, OR if critical cross-story concerns exist:**

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
