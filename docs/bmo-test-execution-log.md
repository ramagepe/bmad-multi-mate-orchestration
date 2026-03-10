# BMO Test Execution Log

**Test Plan:** `docs/bmo-first-test-plan.md`
**Project Under Test:** CardTrader API (test product for BMO validation)
**Start Date:** 2026-03-10

---

## Quick Context for New Agents

**What is this?** We're running BMO (BMAD Multi-Mate Orchestration module) for the first time against a real set of BMAD artifacts to validate that the orchestration workflows actually work. The test product is "CardTrader" — a trading card marketplace API.

**What is BMO?** The orchestration layer of BMAD that dispatches parallel sub-agents via the Task tool. It has 4 workflows: [OI] intellectual refinement, [OD] implementation with git worktrees, [OP] pipeline (OI then OD), [OR] recovery. See `_bmad/bmo/workflows/` for all specs.

**What artifacts exist?** A complete set of BMAD planning and implementation artifacts for CardTrader lives in `test-fixtures/`. These include a PRD (31 FRs, 10 NFRs), architecture doc, epics file (4 epics, 18 stories), and 18 individual story files — all in full BMAD format with Given/When/Then ACs, Tasks/Subtasks, Dev Notes, and File Lists.

**What's the test plan?** Run `check-implementation-readiness` + `sprint-planning` (BMM) then [OI] (intellectual refinement) then [OD] (implementation) against Epic 1 (5 stories). This is a "vanilla run" — no Superpowers patterns except SUBAGENT-STOP safety guard. Full details in `docs/bmo-first-test-plan.md`.

**Current status (2026-03-11):** Phase 0 (setup) ✅ DONE. Phase 1 (BMM pre-workflows) ✅ DONE. Phase 2 ([OI] intellectual refinement — Epic 1) ✅ DONE — 5/5 stories created. Post-run review: ALL 9 improvement observations resolved (I-1–I-7 fixed, I-8–I-9 closed as stale). Phase 2 ([OI] Epic 2) ✅ RUN COMPLETE — session "Epic 2 overview", awaiting analysis. **Next action: Analyze Epic 2 [OI] run results, then propagate src/ changes + run Epic 3/4 or proceed to [OD].**

---

## Key Decisions Log

### 2026-03-10 — Planning Session (Party Mode)

**Participants:** Rami (PO), Tito (Orchestrator), Amelia (Dev), Bob (SM)

**Critical correction by Rami:** The team initially planned to jump straight to [OD] implementation, skipping the intellectual refinement phase entirely. Rami caught this — the stories were generated but never refined or cross-validated by agents. The correct flow is:

```
check-implementation-readiness (BMM) → [OI] refine stories → [OD] implement
```

Not:
```
[OD] implement directly  ← WRONG, skips refinement
```

**Decisions made:**

1. **Repo strategy:** Separate local repo for CardTrader (`~/code/cardtrader`). Simulates real project. Worktree paths match story File Lists. BMAD installed inside CardTrader repo.

2. **Execution order:** Phase 1 `check-implementation-readiness` → Phase 2 [OI] intellectual → Phase 3 [OD] implementation. Each phase in a clean agent session.

3. **Vanilla run:** No Superpowers patterns except SUBAGENT-STOP (safety guard). Establishes baseline for comparison. Superpowers patterns added in second run based on observed pain points.

4. **Advisory pre-check:** BMO step-01 (both OI and OD) will check for existence of `implementation-readiness-report-*.md` and surface findings inline. Non-blocking — informational only.

5. **Why vanilla first (team consensus):** Without a baseline, we can't distinguish between "this Superpowers pattern solved a real problem" vs "this pattern added token overhead for a problem we never had." Context window pressure is the key unknown — extra prompt tokens per sub-agent might HURT if context is already tight.

6. **Dependency analysis for Epic 1:**
   - Intellectual mode: softer constraints. Batch 1 [1.1, 1.2, 1.3] in parallel, Batch 2 [1.4, 1.5]
   - Implementation mode: strict. Sequential batches with merges to dev between each: [1.1] → merge → [1.2, 1.3] → merge → [1.4] → merge → [1.5]

7. **Story size vs context window hypothesis (Rami, pre-run):** If a story is large enough that it inflates the sub-agent's context window significantly, output quality may degrade — missed ACs, shallow implementation, loss of coherence. Neither OI nor OD workflows currently evaluate story size or recommend splitting. Decision: do NOT add story-splitting logic before vanilla run (would contaminate baseline), but DO track per-story context metrics to validate or invalidate this hypothesis. If data confirms the correlation, story-splitting becomes a candidate feature for the second run. Team consensus: the risk is real in general (especially [OD] where code reads compound the context), but NOT for Epic 1 specifically (stories are 3-5 KB, well within capacity).

---

## Pre-Run Hypotheses

### H1: Story Size → Context Pressure → Quality Degradation

**Hypothesis:** Larger stories produce lower-quality sub-agent output because context window saturation reduces coherence and AC coverage.

**What to measure per sub-agent dispatch (both [OI] and [OD]):**

| Metric | How to capture | Notes |
|--------|---------------|-------|
| Story input size (bytes) | `wc -c` on story file | Baseline before refinement |
| Story input size (lines) | `wc -l` on story file | Correlate with bytes |
| AC count | Count Given/When/Then blocks | Complexity proxy |
| Task/subtask count | Count task items in story | Work volume proxy |
| File List entry count | Count files in story's File List | Scope proxy ([OD] only) |
| Refined story output size | `wc -c` on output artifact | Growth factor = output/input |
| AC coverage (self-reported) | From sub-agent's processing_report | May be inflated |
| AC coverage (human-verified) | Manual spot-check post-run | Ground truth |
| Output quality assessment | Human 1-5 rating post-run | Subjective but useful |

**Epic 1 baseline measurements (pre-run — now obsolete):**

> These were the sizes of the 18 pre-existing story files that were DELETED before the creation test. In creation mode, input is epic outlines from `epics.md` (~20–30 lines each), not story files. These measurements are kept for reference only.

| Story | Bytes | Lines | ACs | Tasks |
|-------|-------|-------|-----|-------|
| 1-1 monorepo setup | 5,180 | 154 | — | — |
| 1-2 drizzle sqlite | 3,156 | 97 | — | — |
| 1-3 hono bootstrap | 3,567 | 110 | — | — |
| 1-4 vitest biome | 2,879 | 93 | — | — |
| 1-5 dev scripts | 2,613 | 93 | — | — |

**Current state in BMO workflows:** No step evaluates story size. step-01 readiness check evaluates ACs, clarity, scope, dependencies. step-02 fan-out has `estimated_complexity` but it's about logical complexity, not document size. Sub-agent contracts have no instruction to propose splitting.

**Decision:** Observe in vanilla run. If correlation confirmed → add story-splitting advisory to step-01 and/or step-02 for second run.

**Epic 1 Result (creation mode):** No context pressure observed. Input was epic outlines (~20–30 lines per story in epics.md), output was 9.7–14.8 KB per story. All 5 stories rated 4.0–5.0 quality. H1 NOT triggered for Epic 1 — stories are small enough. Will need larger epics (Epic 2 has more stories) and [OD] implementation mode (code reads compound context) to properly test this hypothesis.

---

## Execution Progress

### Phase 0: Setup

| Step | Status | Notes |
|------|--------|-------|
| 0.1a Change A: Advisory pre-check in step-01 files | ✅ DONE | Added `### 0. Pre-flight` section to OI + OD step-01. Advisory-only, non-blocking. |
| 0.1b Change B: SUBAGENT-STOP in sub-agent contracts | ✅ DONE | Added SUBAGENT-STOP to 5 contract files: OI step-03, OI step-05, OD step-04, OD step-06, OD step-08. |
| 0.2 Create CardTrader repo | ✅ DONE | `~/code/cardtrader`, `git init` |
| 0.3 Install BMAD in CardTrader repo | ✅ DONE | Installed via BMAD installer with custom BMO module. First install missed Change A (installer overwrote `src/`). Re-applied Change A to `src/`, re-installed — both changes confirmed. |
| 0.4 Copy artifacts to CardTrader repo | ✅ DONE | 3 planning artifacts (prd, architecture, epics) + 18 stories. All present. |
| 0.5 Initial commit + dev branch | ✅ DONE | Commit `85b6fd6`, branch `dev` active, working tree clean. |

### Phase 1: BMM Pre-Workflows

| Step | Status | Notes |
|------|--------|-------|
| check-implementation-readiness | ✅ DONE | Run interactively with SM (Bob). Report generated. |
| sprint-planning | ✅ DONE | Run interactively with SM (Bob). `sprint-status.yaml` generated. |

### Phase 2: [OI] Intellectual Refinement — Epic 1

**Sessions:** "Tito Test - OI - Epic 1" (ses_326ba10e6ffeEy8ivXq0f2FIRk) → "Epic 1 overview" (ses_32587a63affe1iJmn3BAn7ukjn, 11 sub-sessions)

| Step | Status | Notes |
|------|--------|-------|
| Trigger Tito with "OI Epic 1" | ✅ DONE | First attempt exposed OBS-2/3/4. Second attempt confirmed pre-flight + auto-discovery. Third session exposed OBS-6/7/8 — created story-creation-panel + story-refinement-panel. Fourth session (Epic 1 overview) completed full end-to-end run. |
| Step 1: Readiness check | ✅ DONE | Auto-discovered architecture, PRD, readiness report, sprint-status. 5/5 stories classified as CREATION candidates (OUTLINE_READY). Clean dependency tree, no cross-story concerns. |
| Step 2: Fan-out planning | ✅ DONE | Applied INTELLECTUAL MODE CONTEXT + CREATION MODE MAXIMIZE PARALLELISM. Single batch of 5 (all parallel). `previous_story_path: null` on all. All deps code-level only. |
| Step 3: Sub-agent dispatch | ✅ DONE | 5 sub-agents dispatched in parallel. Template resolution perfect (no raw `{if}` tags). Contracts clean with absolute paths. All 5 completed successfully. |
| Step 4: Output collection | ✅ DONE | 5/5 artifacts verified on disk. File sizes 9.7–14.8 KB. epics.md integrity confirmed (mtime unchanged). AC spot-check passed all 5. |
| Step 5: Cross-validation | ✅ DONE | Found 3 critical + 4 major + 2 minor issues. Cross-validator dispatched as sub-agent, produced exhaustive 394-line analysis. Report persisted to `_orchestration/cross-validation-report.md` (37 lines — condensed). See OBS-9. |
| Step 6: Corrective loop | ✅ DONE | 1 iteration (of max 3). 4 stories corrected via sub-agents (1.2, 1.3, 1.4, 1.5). 3 minor fixes applied directly by orchestrator. Targeted re-cross-validation confirmed all resolved. Circuit breaker NOT triggered. |
| Step 7: Human escalation | ✅ DONE | No escalation needed — all issues auto-resolved. NOTE: Used old step-05 that asked human for [C/M/F/X] choice; new auto-resolution logic was in `src/` but NOT propagated to CardTrader. See OBS-10. |
| Step 8: Summary report | ✅ DONE | Full execution report with auto-resolved decisions audit trail. Output contract YAML with all 5 artifacts. |

### Phase 2b: [OI] Intellectual Refinement — Epic 2

**Session:** "Epic 2 overview" (ses_32505a2d2ffez6LuKoP5d0LPrv)

**Pre-run changes applied to `src/`:** I-6 (architecture quality advisory), I-7 ({output_path} audit), OBS-13 (auto-proceed gates). Changes propagated to CardTrader before this run.

| Step | Status | Notes |
|------|--------|-------|
| Step 1: Readiness check | ✅ DONE | — |
| Step 2: Fan-out planning | ✅ DONE | — |
| Step 3: Sub-agent dispatch | ✅ DONE | — |
| Step 4: Output collection | ✅ DONE | — |
| Step 5: Cross-validation | ✅ DONE | — |
| Step 6: Corrective loop | ✅ DONE | — |
| Step 7: Human escalation | ✅ DONE | — |
| Step 8: Summary report | ✅ DONE | — |

**Detailed analysis pending** — session complete, results to be reviewed in next analysis session.

### Phase 3: [OD] Implementation

| Step | Status | Notes |
|------|--------|-------|
| Not started — depends on Phase 2 completion | NOT STARTED | — |

---

## Observations & Pain Points

### OBS-1: Module source vs installed copy confusion (Phase 0 — Setup)

- **What happened:** Change A (advisory pre-check) was applied to `_bmad/bmo/` (installed copy) and `src/` (module source). User then ran `rm -rf _bmad/` and re-installed BMAD. The installer regenerated `_bmad/bmo/` from `src/`, but it also overwrote `src/` with a clean copy from the installer registry — wiping Change A from `src/`. Change B (SUBAGENT-STOP) survived for unknown reasons (possibly different installer behavior per file, or timing). Change A had to be re-applied to `src/` and the module re-installed.
- **What was expected:** Edits to `src/` should persist across installer runs, since `src/` is the developer's source for the custom module.
- **Severity:** friction
- **Potential solution:** 
  - **Immediate (process):** Always edit `src/` first, verify changes are in `src/` before running installer, and verify propagation to `_bmad/` after install. Added to `CLAUDE.md` as a project rule.
  - **Long-term (installer):** Installer should NOT overwrite `src/` if it already exists and has local modifications. Or: installer should warn before overwriting modified source files.
- **Category:** external tool (installer behavior) + process

### OBS-2: Step-01 skipped pre-flight check entirely (Phase 2 — First OI attempt)

- **What happened:** When [OI] was triggered with just "OI", the orchestrator jumped straight to asking the user for `epic_path` and `stories` inputs. It did NOT execute `### 0. Pre-flight: Implementation Readiness Report Check` — the advisory section we added in Change A. No readiness report advisory was shown.
- **What was expected:** The agent should execute steps in sequential order: 0 → 1 → 2 → etc. Step 0 should have searched for `implementation-readiness-report-*.md`, not found it, and displayed the advisory message before proceeding to input validation.
- **Severity:** friction (advisory is non-blocking, but if agents skip numbered steps, that's a systemic issue)
- **Root cause:** The `### 0.` section lacked a `🛑 MANDATORY` marker. The mandatory execution rules only said "NEVER skip readiness validation" which the agent interpreted as referring to the main validation (step 1+), not the pre-flight. Fixed by adding explicit `🛑 ALWAYS execute steps in EXACT order: 0 → 1 → 2` to mandatory rules and `🛑 MANDATORY` marker to step 0 heading.
- **Fix applied:** Yes — reinforced in both OI and OD step-01.
- **Category:** BMO-native (step file clarity)

### OBS-3: Orchestrator asked user for epic path instead of auto-discovering (Phase 2 — First OI attempt)

- **What happened:** The orchestrator asked the user to provide `epic_path` manually. The project has a single `epics.md` at `_bmad-output/planning-artifacts/epics.md` which is the standard BMAD convention.
- **What was expected:** The orchestrator should auto-discover the epic file from `{output_folder}/planning-artifacts/epics.md` since that's the convention. Only ask the user if multiple or no epic files are found.
- **Severity:** friction
- **Root cause:** Step-01 input contract had bare placeholders (`epic_path: "{epic_path}"`) with no auto-discovery logic. The step told the agent to validate inputs exist but not how to find them.
- **Fix applied:** Yes — added `### 1a. Resolve epic_path` with auto-discovery sequence in both OI and OD step-01.
- **Category:** BMO-native (step file gap)

### OBS-4: Orchestrator asked user for story file paths (Phase 2 — First OI attempt)

- **What happened:** The orchestrator asked the user to provide a list of story file paths. The 5 stories for Epic 1 already exist at `_bmad-output/implementation-artifacts/stories/1-*.md`.
- **What was expected:** The orchestrator should read the epic file, determine the epic number, and glob for matching stories (e.g., `1-*.md` for Epic 1). Only ask the user if no matching stories are found.
- **Severity:** friction
- **Root cause:** Same as OBS-3 — step-01 had no auto-discovery logic for stories.
- **Fix applied:** Yes — added `### 1b. Resolve stories` with auto-discovery sequence in both OI and OD step-01.
- **Category:** BMO-native (step file gap)

### OBS-5: Pre-flight only checked readiness report, not sprint-status (Phase 0 — Setup review)

- **What happened:** Rami pointed out that BMO's pre-flight should check for BOTH prerequisite BMM artifacts: the implementation readiness report AND the sprint-status.yaml. The original Change A only checked for the readiness report.
- **What was expected:** BMO should verify the full standard BMAD flow was followed: `check-implementation-readiness` → `sprint-planning` → BMO. Both produce artifacts BMO should be aware of.
- **Severity:** minor (sprint-status.yaml is not consumed by BMO directly, but its absence indicates the standard flow wasn't followed)
- **Root cause:** Original pre-flight spec only considered one BMM prerequisite. The sprint-planning step was not part of the initial test plan analysis.
- **Fix applied:** Yes — expanded pre-flight to `### 0. Pre-flight: BMM Artifact Checks` with sub-steps 0a (readiness report), 0b (sprint-status.yaml), 0c (combined summary). Both OI and OD step-01 updated.
- **Category:** BMO-native (step file gap)

---

## Metrics (Vanilla Baseline)

_To be filled after each phase completes._

### Aggregate Metrics

| Metric | Phase 1 | Phase 2 (Epic 1) | Phase 3 |
|--------|---------|---------|---------|
| Wall time | — | ~10 min (~4 min dispatch + ~3 min cross-val + ~3 min corrective) | — |
| Sub-agent dispatches | N/A | 11 (5 creation + 1 cross-val + 4 corrective + 1 targeted re-validation) | — |
| Corrective loops | N/A | 1 (of max 3) | — |
| Stories completed | N/A | 5/5 | — |
| Stories failed | N/A | 0 | — |
| Cross-validation issues | N/A | 7 (3 critical + 4 major) → ALL resolved | — |
| Human escalation | N/A | 0 | — |
| Total output | N/A | 61,232 bytes / 1,321 lines across 5 stories + 2,588 byte cross-val report | — |
| Context pressure observed | — | None (stories 2.6–5.2 KB input, well within capacity) | — |

### Per-Story Context Metrics (H1 Tracking)

_Fill during [OI] Phase 2 and [OD] Phase 3. Measures input/output size per sub-agent to validate H1._

#### Phase 2: [OI] Intellectual Refinement — Epic 1

> **Mode: CREATION** — Input was epic outlines from `epics.md`, not pre-existing story files. The "Input" column below reflects the outline size in the epic, not a story file. The pre-run baseline measurements (5,180 / 3,156 / etc.) were for the 18 pre-existing story files that were deleted before the creation test.

| Story | Output (bytes) | Output (lines) | ACs | Tasks | Subtasks | Corrections | Quality (1-5) | Notes |
|-------|---------------|----------------|-----|-------|----------|-------------|---------------|-------|
| 1-1 monorepo setup | 14,810 | 358 | 4 | 6 | 23 | 0 (anchor) | 5.0 | Exceptional. Complete code snippets, .js extensions, directory tree with ⚠️ markers, [Source: §] refs |
| 1-2 drizzle sqlite | 12,617 | 231 | 7 | 10 | — | 1 | 4.5 | Post-corrective: VERIFY vs CREATE differentiated, .js extensions added, PaginationMeta resolved |
| 1-3 hono bootstrap | 12,515 | 259 | 6 | 7 | — | 1 | 4.5 | Post-corrective: health.test.ts added (AC6+Task5), .js extensions, workspace:\* → \*, app.request() testing |
| 1-4 vitest biome | 9,745 | 220 | 4 | 7 | 17 | 1 | 4.0 | Post-corrective: vitest.workspace.ts standardized on config-file glob |
| 1-5 dev scripts | 11,545 | 253 | 5 | 8 | — | 1 | 4.0 | Post-corrective: tsx as explicit devDependency, end-to-end verification task |
| **TOTALS** | **61,232** | **1,321** | **26** | **38+** | | | **4.4 avg** | |

#### Phase 2 Cross-Validation Issues (Epic 1)

| # | Severity | Issue | Stories | Resolution |
|---|----------|-------|---------|------------|
| C1 | CRITICAL | `health.test.ts` has no owner — no story creates it, each points to another | 1.3, 1.4, 1.5 | Added to Story 1.3 (AC6 + Task 5). Source: architecture.md § Testing Patterns |
| C2 | CRITICAL | `.js` extensions missing in imports — NodeNext requires them, Stories 1.2/1.3 omit | 1.1, 1.2, 1.3 | Standardized .js in all imports. Source: NodeNext module resolution rules |
| C3 | CRITICAL | `workspace:*` (pnpm protocol) in Story 1.3 — project uses npm, needs `"*"` | 1.1 vs 1.3 | Fixed to `"*"`. Source: Story 1.1 as authoritative |
| M1 | MAJOR | Triple creation of shared files — types/api.ts, validators/index.ts, constants/index.ts created in 3 stories | 1.1, 1.2, 1.3 | Story 1.1 sole owner. 1.2/1.3 → VERIFY/MODIFY |
| M2 | MAJOR | PaginationMeta inconsistency — separate interface (1.2) vs inline (1.1, 1.3) | 1.1, 1.2, 1.3 | Removed separate interface, use inline. Source: architecture.md § API Response |
| M3 | MAJOR | vitest.workspace.ts glob mismatch — `['packages/*']` (1.4) vs `['packages/*/vitest.config.ts']` (1.5) | 1.4, 1.5 | Standardized on config-file glob (per-package configs exist) |
| M4 | MAJOR | `tsx` not installed — Story 1.5 uses `tsx watch` but no story adds it as devDependency | 1.5 | Added as devDependency to api package |

All 7 issues resolved in 1 corrective iteration. Zero human escalation.

#### Phase 3: [OD] Implementation

| Story | Input (bytes) | Input (lines) | ACs | Tasks | File List Entries | Output Files | Tests Passed | AC Coverage (self) | AC Coverage (human) | Quality (1-5) |
|-------|--------------|---------------|-----|-------|------------------|--------------|--------------|--------------------|--------------------|----|
| 1-1 | — | — | — | — | — | — | — | — | — | — |
| 1-2 | — | — | — | — | — | — | — | — | — | — |
| 1-3 | — | — | — | — | — | — | — | — | — | — |
| 1-4 | — | — | — | — | — | — | — | — | — | — |
| 1-5 | — | — | — | — | — | — | — | — | — | — |

---

## Observations & Pain Points (continued)

### OBS-6: Tito didn't extract epic number from user's original prompt (Phase 2 — OI attempt)

- **What happened:** User said "Refine stories for Epic 1" — Tito started step-01 but then asked the user again for the epic number, despite it being present in the original prompt.
- **What was expected:** The orchestrator should parse the user's initial request and carry the epic number into step-01 without re-asking.
- **Severity:** friction
- **Root cause:** step-01 input parsing doesn't instruct the orchestrator to extract parameters from the user's original prompt — it treats inputs as explicit parameters to be collected.
- **Fix applied:** Not yet — noted for post-vanilla improvements. Low priority since the user just re-states it.
- **Category:** BMO-native (step file gap)

### OBS-7: Tito applied implementation-level dependency ordering to intellectual mode (Phase 2 — OI batching) — RESOLVED

- **What happened:** Tito analyzed Epic 1's 5 stories and produced 4 sequential batches based on code-level dependencies (e.g., Story 1.2 "depends on 1.1 for monorepo structure"). This is correct for [OD] implementation mode but overly conservative for [OI] intellectual mode, where sub-agents refine/create *documents*, not code.
- **What was expected:** In intellectual mode, most stories can be processed in parallel because document refinement doesn't require compiled outputs. Only genuine *content-level* dependencies (where Story B's processing requires Story A's *processed output*) should force sequencing.
- **Severity:** efficiency (4 batches instead of 2 = 2x longer wall time)
- **Root cause:** step-02 had no distinction between intellectual and implementation modes for dependency analysis.
- **Fix applied:** Yes — Added "INTELLECTUAL MODE CONTEXT" block to step-02 section 3 explaining the distinction between code-level and content-level dependencies. Result: 4 batches → 2 batches (3 stories + 2 stories).
- **Category:** BMO-native (step file gap)

### OBS-8: create-story workflow doesn't work as sub-agent workflow (Phase 2 — Architecture Discovery)

- **What happened:** During design of the story creation flow, we discovered that the BMM `create-story` workflow is interactive — it uses `<ask>` tags for user input, performs web research, updates sprint-status.yaml, and follows an interactive dialog pattern. It cannot be invoked by a sub-agent without human interaction.
- **What was expected:** Sub-agents need a non-interactive workflow for story creation. The original step-02 routing table had no entry for "story has no file — needs creation."
- **Severity:** blocker (story creation impossible without new panel)
- **Root cause:** The OI workflow was originally designed only for *refinement* of existing stories. The test revealed that its PRIMARY use case should be story *creation* — generating story files from epics.md outlines. The 18 pre-existing story files in CardTrader were generated during setup; they shouldn't have been there for a proper OI test.
- **Fix applied:** Yes — Created `story-creation-panel.md` (278 lines) as a non-interactive, sub-agent-compatible story creation workflow. Created `story-refinement-panel.md` (259 lines) for the refinement case. Updated step-02 routing table and step-03 dispatch contract to support both task types.
- **Category:** BMO-native (design gap — major)

### OBS-9: Cross-validation report persisted to disk is much thinner than sub-agent analysis (Phase 2 — Epic 1 run)

- **What happened:** The cross-validator sub-agent produced a 394-line exhaustive analysis covering file creation conflicts, content contradictions, import extension issues, dependency protocol mismatches, coverage gaps, duplicate requirements, terminology inconsistencies, and dependency issues. The report persisted to `_orchestration/cross-validation-report.md` is only 37 lines — a condensed YAML summary.
- **What was expected:** The persisted report should capture enough detail to be useful for audit and future reference without re-running the cross-validator.
- **Severity:** minor (the orchestrator has the full analysis in context from the sub-agent output — it's the persistent artifact that's thin)
- **Root cause:** step-05 cross-validation doesn't specify how much of the sub-agent's analysis to persist vs summarize. The orchestrator chose to write only the structured YAML report, discarding the narrative analysis.
- **Potential fix:** step-05 or step-08 should instruct the orchestrator to persist the full cross-validation analysis (or at minimum the findings + recommendations) to the report file, not just the YAML summary.
- **Category:** BMO-native (step file gap — output specification)

### OBS-10: step-05 asked human for corrective loop choice despite all issues being auto-resolvable (Phase 2 — Epic 1 run)

- **What happened:** After cross-validation found 7 issues, Tito presented the user with `[C] Entrar al corrective loop / [M] Resolución manual / [F] Force proceed / [X] Abortar`. The user chose [C]. All 7 issues were then resolved automatically in 1 iteration with zero human input needed for any individual fix.
- **What was expected:** The updated step-05 in `src/` classifies issues as AUTO-RESOLVABLE vs NEEDS-HUMAN. All 7 of these issues were AUTO-RESOLVABLE (clear authoritative source, unambiguous fix). The orchestrator should have entered the corrective loop automatically without asking.
- **Severity:** friction (1 unnecessary human interaction)
- **Root cause:** The CardTrader installed copy of step-05 still had the old version. The auto-resolution logic was added to `src/` but never propagated to CardTrader for this run.
- **Fix:** Propagate updated step-05 from `src/` to CardTrader before Epic 2 run.
- **Category:** process (known gap — intentionally deferred for this run)

### OBS-11: Cross-validation catches are exactly the type of issues parallel creation produces (Phase 2 — Epic 1 run)

- **What happened:** All 7 cross-validation issues fall into a predictable pattern: when 5 sub-agents create stories in parallel from the same epic outlines, they each independently interpret shared resources (file ownership, import conventions, dependency protocols, test ownership) without seeing each other's output. The cross-validator caught all of these.
- **Significance:** This validates the OI workflow design — parallel creation + cross-validation + corrective loop is the RIGHT architecture. Forcing sequential creation to avoid these issues would be slower AND wouldn't guarantee consistency (agents still might diverge). The cross-validator is purpose-built for this.
- **Takeaway:** The cross-validator is the MVP of the OI workflow. Its cost (~3 min) is negligible compared to the time saved by parallel creation (~4 min for 5 stories vs ~20 min sequential). The corrective loop cost (~3 min) is the price of parallel divergence, and it's worth paying.
- **Category:** validation (positive — architecture confirmed)

### OBS-12: Story quality rating ⭐⭐⭐⭐–⭐⭐⭐⭐⭐ with anti-generic mandate working (Phase 2 — Epic 1 run)

- **What happened:** All 5 stories received 4.0–5.0 human quality ratings. Key quality markers observed: complete code snippets with exact file content, `.js` extensions in imports, `"*"` for npm workspaces, API types defined inline with exact interfaces, shared file stubs with comments explaining when populated, directory tree with ⚠️ markers, `[Source: archivo § sección]` references, concrete verification commands, dev notes with reasoned decisions.
- **Significance:** The story-creation-panel's anti-generic mandate (with BAD/GOOD examples) is working. Sub-agents produced specific, actionable stories — not vague "follow best practices" filler. The dev agent (Amelia) confirmed: "puedo agarrar Story 1.1 ahora mismo y implementarla sin abrir un chat."
- **Category:** validation (positive — quality mandate confirmed)

### Key Architectural Decisions (OBS-6–8 session)

1. **Story creation is the primary OI use case.** Stories in epics.md are outlines; the OI workflow's main job is to expand them into full implementation-ready story files. Refinement is the secondary case (when stories already exist but need strengthening).

2. **Two separate panels, not one.** Creation and refinement have different phase structures (3-phase vs 2-phase), different inputs (outline vs existing story), and different validation concerns. Shared multi-perspective validation at the end.

3. **No sandbox directories.** Sub-agents write story files directly to `{stories_output_path}` (= `{output_folder}/implementation-artifacts/stories/`). No per-story subdirectories, no intermediate sandbox, no promotion step. Simplifies the flow and matches BMM conventions.

4. **Wendy (workflow-builder) consulted before implementation.** Two-pass approach: consult Wendy for design → review her output → implement. This caught issues early (panel structure, output format, template embedding) before they became code.

5. **CardTrader story files deleted for proper creation test.** The 18 pre-existing story files were removed from `_bmad-output/implementation-artifacts/stories/` so the OI test runs creation mode, not refinement mode.

6. **Concretion requirements in story-creation-panel.** Anti-generic mandate with BAD/GOOD examples ensures sub-agents produce specific, actionable stories — not vague "follow best practices" filler. This was extracted from lessons learned in the checklist.md disaster prevention items.

---

## Post-Run Review Session — Improvement Observations

### Review methodology

Team review (party-mode) of Epic 1 run results. 9 improvement observations identified (I-1 through I-9). Each reviewed one-by-one with the full agent team.

### I-1: step-05 old version in CardTrader (auto-resolution not propagated)

- **Status:** ✅ Deferred to next session — propagate step-05/06/08 from `src/` to CardTrader before Epic 2
- **No code change needed** — already fixed in `src/`, just not copied to CardTrader

### I-2: Cross-validation report persisted to disk is too thin (37 lines vs 394)

- **Status:** ✅ FIXED
- **Root cause:** step-05 had no instruction to persist the sub-agent's full analysis. step-08 said "write report" but didn't specify what content.
- **Fix:** Added new section 5 "Persist Full Cross-Validation Analysis" to step-05 — instructs orchestrator to write COMPLETE analysis verbatim (narrative + YAML) immediately after receiving sub-agent output. step-08 section 7 changed from "write report" to "verify report exists on disk."
- **Files changed:** `src/step-05-cross-validation.md` (+25 lines, sections renumbered 5→10), `src/step-08-summary-report.md` (section 7 simplified)

### I-3: Story 1.5 overlap with 1.4 on root package.json scripts

- **Status:** ✅ FIXED (upstream prevention + downstream detection)
- **Root cause:** Cross-validator SAW the overlap (line 190-195 of analysis) but classified it as "sequential modification, not a true conflict" and didn't elevate it to a formal issue. The cross-validator contract didn't include file ownership clarity as a validation criterion.
- **Fix (upstream — creation panel):** Expanded File List section with explicit CREATE/VERIFY/MODIFY/ADD action definitions. Only ONE story may CREATE a shared file. Others must use VERIFY/MODIFY/ADD with `[Owner: Story X.Y]` reference.
- **Fix (downstream — cross-validator):** Added criterion 6 "File Ownership Clarity" to the cross-validator contract. Added `file_ownership_issues` section to YAML report format.
- **Files changed:** `src/story-creation-panel.md` (+8 lines), `src/step-05-cross-validation.md` (+12 lines)

### I-4: Hardcoded dependency versions (training data may be outdated)

- **Status:** ✅ FIXED
- **Root cause:** Sub-agents invented specific version numbers (e.g., `drizzle-orm@^0.45.1`) from training data. Architecture doc says `latest` without resolving. Sub-agent filled in a concrete but potentially outdated number.
- **Decision:** Version pinning belongs in the architecture workflow (upstream, BMM), not in OI. OI should use architecture doc versions literally.
- **Fix:** Added "Dependency Versions" rule to creation panel: use architecture doc version literally; if `latest`/omitted → use `latest` + dev note warning; NEVER invent versions from memory. Reinforced in Tasks/Subtasks section.
- **Files changed:** `src/story-creation-panel.md` (+5 lines)
- **Future:** Architecture workflow (BMM) should pin versions with web research at technology selection time. Not BMO scope.

### I-5: Story 1.4 vague about Biome fix expectations

- **Status:** ✅ FIXED
- **Root cause:** Story says "fix any Biome errors" without specifying what to expect. Sub-agent can't predict linting errors on code that doesn't exist yet.
- **Fix:** Added linter verification pattern to creation panel Tasks/Subtasks: "Run `{linter} --write .` to auto-fix, then `{linter}` to verify zero remaining issues." Auto-fix first, analyze only what survives.
- **Files changed:** `src/story-creation-panel.md` (+1 line)

### I-6: Architecture doc as upstream source of issues — pre-scan advisory

- **Status:** ✅ FIXED
- **Proposal:** Add sub-step 0e "Architecture Quality Advisory" to step-01 readiness check. Non-blocking scan of architecture doc for: unresolved versions (`latest` without pin), atypical patterns, incomplete sections. Surfaces warnings but does NOT block the run.
- **Team consensus:** Good idea but avoid scope creep in step-01. Advisory only, not blocking.
- **Resolution:** Implemented as step `0e` in both OI and OD step-01. Runs 5 checks (P1–P5): unresolved versions (`latest`), explicit placeholders (TBD/TODO/FIXME), empty sections, key area coverage (keywords matching creation panel A3 areas), structure section without concrete paths. Advisory-only, never blocks. Extends existing `0d` (architecture discovery) — reads the already-loaded file, zero additional latency.
- **Files changed:** `src/workflows/orchestrate-intellectual/steps/step-01-readiness-check.md`, `src/workflows/orchestrate-implementation/steps/step-01-readiness-check.md`

### I-7: `{output_path}` vs `{stories_output_path}` references in step-05/step-06

- **Status:** ✅ FIXED
- **Context:** step-05 section 2 `validation_context` YAML still references `{output_folder}/{story-id-1}/` per-story subdirectory structure. This was fixed in step-02/step-03 but not in step-05/step-06.
- **Resolution:** Full audit of ALL `{output_path}` references across the entire OI workflow. Replaced with `{stories_output_path}/{story_key}.md` (for specific story files) or `{stories_output_path}` (for the directory). Changes span step-04 (verification commands), step-05 (contract + validation_context YAML), step-06 (corrective contract + verification), step-07 (escalation display + manual fix + drop), step-08 (artifact inventory + story detail + pending actions), and workflow.md (sub-agent contract + security rules). Zero `{output_path}` references remain in the OI workflow.
- **Files changed:** `src/workflows/orchestrate-intellectual/steps/step-04-output-collection.md`, `step-05-cross-validation.md`, `step-06-corrective-loop.md`, `step-07-human-escalation.md`, `step-08-summary-report.md`, `workflow.md`

### I-8: Tito didn't extract epic number from original prompt

- **Status:** ✅ CLOSED — Stale observation, no longer relevant
- **Original OBS-6:** This was from an earlier session (pre-Epic 1 run), not from the successful Epic 1 run. In the Epic 1 run, user typed "OI Epic 1" and Tito extracted it correctly. The observation may be stale.
- **Resolution:** Verified against OpenCode session `ses_32587a63affe1iJmn3BAn7ukjn` ("OU Test - Epic 1"). First user message was `"OI Epic 1"` — Tito parsed it correctly on the first response: *"the user said 'OI Epic 1' — that matches [OI] with 'Epic 1' as the target argument."* Auto-discovered epic_path, extracted Epic 1, searched stories with prefix `1-`. No re-asking. OBS-6 was from a prior session where user said only `"OI"` without specifying the epic — re-asking was correct behavior in that case.

### I-9: Batching concerns for epics >5 stories

- **Status:** ✅ CLOSED — Confirmed always 1 batch; future batching is an observation, not a fix
- **Correction:** Rami is right. In the successful Epic 1 run, it was always 1 batch of 5 (all parallel). The "2 batches (3+2)" reference was from the OBS-7 session BEFORE the CREATION MODE MAXIMIZE PARALLELISM fix was applied. After that fix, Epic 1 was always 1 batch.
- **Resolution:** Verified against OpenCode session `ses_32587a63affe1iJmn3BAn7ukjn`. Fan-out planning applied `CREATION MODE = MAX PARALLELISM`, assigned `previous_story_path: null` to all 5 stories, dispatched 1 batch of 5 in parallel. The "2 batches (3+2)" was from an earlier OBS-7 session before the fix. For Epic 2 with more stories, if `max_parallel_agents` forces batching, that's expected behavior — not a bug. Will observe in Epic 2 run.

---

## Current Status (2026-03-11)

**Where we are:** Phase 2 [OI] for Epic 1 is COMPLETE. All 9 post-run improvement observations resolved (I-1–I-7 fixed, I-8–I-9 closed). Phase 2b [OI] for Epic 2 has been RUN — session complete, detailed analysis pending.

**Artifacts produced — Epic 1 (CardTrader `_bmad-output/`):**
- `implementation-artifacts/stories/1-1-monorepo-setup.md` (14,810 B, 358 lines)
- `implementation-artifacts/stories/1-2-drizzle-sqlite-setup.md` (12,617 B, 231 lines)
- `implementation-artifacts/stories/1-3-hono-app-bootstrap.md` (12,515 B, 259 lines)
- `implementation-artifacts/stories/1-4-vitest-biome-configuration.md` (9,745 B, 220 lines)
- `implementation-artifacts/stories/1-5-dev-scripts-first-test.md` (11,545 B, 253 lines)
- `_orchestration/cross-validation-report.md` (2,588 B, 37 lines)

**Artifacts produced — Epic 2:** Pending analysis of session data.

**Opencode sessions:**
- "Tito Test - OI - Epic 1" (ses_326ba10e6ffeEy8ivXq0f2FIRk) — initial session, pre-flight verification
- "Epic 1 overview" (ses_32587a63affe1iJmn3BAn7ukjn) — full OI run, 11 sub-sessions
- "BMO validation Phase 2 Epic 1 review" (ses_3252715a6ffeRpWFKMEkpPqEvm) — party-mode review session, resolved I-1 through I-9 + OBS-13
- "Epic 2 overview" (ses_32505a2d2ffez6LuKoP5d0LPrv) — [OI] Epic 2 run, awaiting analysis

**Source files modified across all review sessions (`src/`):**
- `src/workflows/orchestrate-intellectual/steps/step-01-readiness-check.md` — [I-6] Architecture Quality Advisory (0e) P1–P5, [OBS-13] auto-proceed gate
- `src/workflows/orchestrate-implementation/steps/step-01-readiness-check.md` — [I-6] Architecture Quality Advisory (0e) + discovery (0d), [OBS-13] auto-proceed gate
- `src/workflows/orchestrate-intellectual/steps/step-02-fan-out-planning.md` — [OBS-13] auto-proceed gate when plan is straightforward
- `src/workflows/orchestrate-intellectual/steps/step-04-output-collection.md` — [I-7] `{output_path}` → `{stories_output_path}`
- `src/workflows/orchestrate-intellectual/steps/step-05-cross-validation.md` — [I-2] full analysis persistence, [I-3] file ownership criterion, [I-7] path fix
- `src/workflows/orchestrate-intellectual/steps/step-06-corrective-loop.md` — [I-7] path fix in contract + verification
- `src/workflows/orchestrate-intellectual/steps/step-07-human-escalation.md` — [I-7] path fix throughout
- `src/workflows/orchestrate-intellectual/steps/step-08-summary-report.md` — [I-2] verify-not-write cross-val, [I-7] path fix
- `src/workflows/orchestrate-intellectual/steps/story-creation-panel.md` — [I-3][I-4][I-5] dependency versions, linter, file ownership
- `src/workflows/orchestrate-intellectual/workflow.md` — [I-7] contract + security rules

**Next actions:**
1. ~~All I-1 through I-9 improvements~~ ✅ RESOLVED
2. ~~OBS-13 auto-proceed gates~~ ✅ FIXED
3. ~~Propagate src/ to CardTrader~~ ✅ DONE (before Epic 2 run)
4. ~~Run [OI] for Epic 2~~ ✅ RUN COMPLETE
5. **Analyze Epic 2 [OI] session results** ← NEXT
6. Decide: run Epic 3/4 [OI], or proceed to [OD] for Epic 1

---

### OBS-13: Unnecessary user prompts when all stories are ready and plan is straightforward (Phase 2 — Epic 2 run)

- **What happened:** When running [OI] for Epic 2 with 4/4 stories OUTLINE_READY, Tito asked the user two unnecessary questions: (1) step-01 decision gate `[A/R/S/X]` when there are no stories to exclude, and (2) step-02 fan-out confirmation `[Y/N/E]` when the plan is a single batch with no complexity.
- **What was expected:** When there's no real decision to make (all stories ready, no critical concerns, single batch, same workflow), the orchestrator should auto-proceed with a visible log message — not stop and ask.
- **Severity:** friction (2 unnecessary human interactions per run)
- **Root cause:** step-01 section 6 and step-02 section 5 always present interactive gates regardless of whether there's a decision to make.
- **Fix applied:** Yes — added auto-proceed conditions to both gates in OI step-01, OI step-02, and OD step-01. Gates still fire when there are PARTIAL/NOT_READY stories, critical cross-story concerns, multiple batches, or mixed workflows. Plan is always displayed; only the prompt is skipped.
- **Files changed:** `src/workflows/orchestrate-intellectual/steps/step-01-readiness-check.md`, `src/workflows/orchestrate-intellectual/steps/step-02-fan-out-planning.md`, `src/workflows/orchestrate-implementation/steps/step-01-readiness-check.md`
- **Category:** BMO-native (step file gap — unconditional gates)

---

_Log started 2026-03-10_
