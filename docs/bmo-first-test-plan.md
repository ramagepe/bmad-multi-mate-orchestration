# BMO First Test Plan — Epic 1 Vanilla Run

**Date:** 2026-03-10
**Status:** Planned — ready for execution
**Decided via:** BMAD Party Mode session with full agent team
**Participants:** Rami (PO), Tito (Orchestrator), Amelia (Dev), Bob (SM)

---

## Objectives

1. **Validate BMO orchestration end-to-end** using real BMAD artifacts against a real product (CardTrader API)
2. **Establish baseline metrics** (vanilla run) before applying Superpowers patterns
3. **Identify pain points** in the current workflow: context window pressure, path resolution, dependency handling, sub-agent behavior
4. **Test three BMO capabilities sequentially**: advisory pre-check, intellectual refinement ([OI]), implementation orchestration ([OD])

---

## Key Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | **Separate local repo** for CardTrader | Simulates real project setup. Worktree paths match story File List entries. No mixing of tool code (BMO) with product code (CardTrader). |
| 2 | **Epic 1** as first test | Greenfield scenario. 5 stories. No shared file conflicts. Tests the full 14-step pipeline without intra-epic complications. |
| 3 | **Branch `dev`** as base | Already created in BMO repo. Will be replicated in CardTrader repo. Standard branch for worktree creation. |
| 4 | **`check-implementation-readiness` as advisory pre-check** | Non-blocking. BMO step-01 checks if the report exists and surfaces critical issues inline. Complements (not duplicates) BMO's own readiness validation. |
| 5 | **Vanilla run first** | Establishes baseline. Prevents confounding variables. Measures where context pressure actually bites. SUBAGENT-STOP is the only exception (safety guard). |
| 6 | **Second run with improvements** after analysis | Apply own improvements + selected Superpowers patterns based on observed pain points. Compare against baseline. |

---

## Execution Sequence

### Phase 0: Setup

#### 0.1 Apply Pre-Run Changes to BMO Step Files

Two minimal changes before any execution:

**Change A: Advisory Pre-Check in step-01 (both workflows)**

Add a new section "### 0. Pre-flight: Implementation Readiness Report Check" at the beginning of the execution sequence in:
- `_bmad/bmo/workflows/orchestrate-intellectual/steps/step-01-readiness-check.md`
- `_bmad/bmo/workflows/orchestrate-implementation/steps/step-01-readiness-check.md`

Behavior:
```
Search: {planning_artifacts}/implementation-readiness-report-*.md

IF found:
  Read report → extract "Overall Readiness Status" + "Critical Issues"
  Display inline:
    IF READY:  "Advisory: Implementation readiness check completed: READY"
    IF NEEDS WORK: "Advisory: Readiness check found issues: [list critical issues]. 
                    Report: {path}. Recommend addressing before proceeding."
  DO NOT BLOCK — proceed to standard step-01 validation

IF not found:
  Display: "Advisory: No implementation readiness report found.
           Recommendation: Run 'check-implementation-readiness' workflow first.
           This is advisory only — you may proceed without it."
  DO NOT BLOCK — proceed to standard step-01 validation
```

**Change B: SUBAGENT-STOP in sub-agent contracts**

Add to all sub-agent contract templates in:
- `_bmad/bmo/workflows/orchestrate-intellectual/steps/step-03-sub-agent-dispatch.md` (document processor contract)
- `_bmad/bmo/workflows/orchestrate-implementation/steps/step-04-implementor-fanout.md` (implementor contract)
- `_bmad/bmo/workflows/orchestrate-implementation/steps/step-06-reviewer-dispatch.md` (reviewer contract)
- `_bmad/bmo/workflows/orchestrate-implementation/steps/step-08-cross-validation.md` (cross-validator contract)
- `_bmad/bmo/workflows/orchestrate-intellectual/steps/step-05-cross-validation.md` (cross-validator contract)

Add under CRITICAL RESTRICTIONS:
```
- SUBAGENT-STOP: You are a SUB-AGENT dispatched for a specific task. 
  Do NOT invoke BMO orchestration workflows. Do NOT dispatch your own 
  sub-agents via Task tool. Complete YOUR task and report back.
```

#### 0.2 Create CardTrader Repo

```bash
mkdir ~/code/cardtrader
cd ~/code/cardtrader
git init
```

#### 0.3 Install BMAD in CardTrader Repo

Option A (preferred): Run BMAD installer if available
Option B (manual): Copy `_bmad/` directory structure from BMO repo

Required modules: core, bmm, bmo (tea optional for first test)

#### 0.4 Copy Artifacts to CardTrader Repo

```bash
# Planning artifacts
mkdir -p _bmad-output/planning-artifacts
cp ~/code/bmad-multi-mate-orchestration/test-fixtures/planning-artifacts/prd.md _bmad-output/planning-artifacts/
cp ~/code/bmad-multi-mate-orchestration/test-fixtures/planning-artifacts/architecture.md _bmad-output/planning-artifacts/
cp ~/code/bmad-multi-mate-orchestration/test-fixtures/planning-artifacts/epics.md _bmad-output/planning-artifacts/

# Implementation artifacts (stories)
mkdir -p _bmad-output/implementation-artifacts/stories
cp ~/code/bmad-multi-mate-orchestration/test-fixtures/implementation-artifacts/stories/*.md _bmad-output/implementation-artifacts/stories/
```

#### 0.5 Initial Commit and Branch

```bash
git add .
git commit -m "initial: BMAD artifacts for CardTrader API"
git checkout -b dev
```

---

### Phase 1: Implementation Readiness Check (BMM)

**Workflow:** `check-implementation-readiness`
**Agent:** SM (Bob) or PM (John)
**Mode:** Interactive (not orchestrated by BMO)
**Duration estimate:** 15-20 minutes

**What it validates (6 steps):**
1. Document discovery — PRD, architecture, epics exist
2. PRD analysis — FRs complete and traceable
3. Epic coverage — all FRs covered by stories
4. UX alignment — N/A for CardTrader (API-only, no UX doc)
5. Epic quality — stories have ACs, dependencies, scope
6. Final assessment — GO/NO-GO

**Expected output:** `_bmad-output/planning-artifacts/implementation-readiness-report-2026-03-10.md`

**Trigger command to agent:**
```
Check implementation readiness for CardTrader API
```

**After completion:**
- Review the report
- Fix any critical issues found
- Commit changes if artifacts were modified

---

### Phase 2: Intellectual Refinement ([OI])

**Workflow:** `orchestrate-intellectual`
**Agent:** Tito (Orchestrator)
**Mode:** Orchestrated parallel (up to 3 sub-agents)
**Duration estimate:** 30-45 minutes

**Trigger — ideal minimal prompt:**
```
Refine the stories for Epic 1: Project Foundation
```

**What Tito should resolve internally:**
- `epic_path` = `_bmad-output/planning-artifacts/epics.md` (convention from config `output_folder`)
- `stories` = glob `_bmad-output/implementation-artifacts/stories/1-*.md` (5 files)
- `parallel_limit` = 3 (from `_bmad/bmo/config.yaml`)
- `output_folder` = `_bmad-output` (from config)

**What [OI] does (8 steps):**
1. Readiness check (+ advisory pre-check for readiness report)
2. Fan-out planning (assign `create-story` workflow to each story, batch by dependencies)
3. Sub-agent dispatch (create-story refinement per story, parallel batches)
4. Output collection (verify artifacts produced)
5. Cross-validation (check coherence across all 5 refined stories)
6. Corrective loop (if cross-validator found issues, max 3 iterations)
7. Human escalation (if circuit breaker exhausted)
8. Summary report

**Epic 1 dependency analysis for batching:**
```
1.1 (Monorepo Setup)       → independent
1.3 (Hono App Bootstrap)   → depends on 1.1 (needs api package)
1.2 (Drizzle + SQLite)     → depends on 1.1 (needs core package)
1.4 (Vitest + Biome)       → depends on 1.3 (needs health test)
1.5 (Dev Scripts)           → depends on 1.4 (needs test runner)

Recommended batches (intellectual mode — refinement, not implementation):
  Batch 1: [1.1, 1.2, 1.3] — all 3 can be refined in parallel 
           (refinement doesn't need code to exist, only story context)
  Batch 2: [1.4, 1.5] — refine after batch 1 outputs exist
```

> **Note:** In intellectual mode, dependency constraints are softer — stories are being refined as documents, not implemented. The cross-validator will catch any dependency issues in step 5.

**Expected output:**
- 5 refined story files in `_bmad-output/` (per story subdirectories)
- Cross-validation report in `_bmad-output/_orchestration/`
- Summary report from step 8

**After completion:**
- Review refined stories
- Review cross-validation findings
- Decide: use refined stories for implementation, or originals if refinement didn't improve them

---

### Phase 3: Implementation Orchestration ([OD])

**Workflow:** `orchestrate-implementation`
**Agent:** Tito (Orchestrator)
**Mode:** Orchestrated parallel with git worktrees
**Duration estimate:** 60-90 minutes

**Trigger — ideal minimal prompt:**
```
Implement the stories for Epic 1: Project Foundation
Base branch: dev
```

**What Tito should resolve internally:**
- `epic_path` = `_bmad-output/planning-artifacts/epics.md`
- `stories` = the refined stories from Phase 2 (or originals)
- `base_branch` = `dev`
- `parallel_limit` = 3 (from config)
- `worktree_base_path` = `.claude/worktrees` (from config)

**What [OD] does (14 steps):**
1. Readiness check (+ advisory pre-check)
2. Shared file snapshot (baseline for mutation detection)
3. Worktree setup (create worktrees + branches, sequential)
4. Implementor fan-out (sub-agents in parallel, batched)
5. Output collection (verify commits, tests, lint)
6. Reviewer dispatch (code review per story)
7. Corrective loop (fix review findings, max 3 iterations)
8. Cross-validation (check coherence across stories)
9. Shared file mutation check (diff worktrees against snapshot)
10. Pre-merge gate (conflict detection, test verification)
11. Human approval gate (per-branch user approval)
12. Push approved branches
13. Cleanup (remove worktrees)
14. Summary report

**Epic 1 dependency analysis for implementation batching:**
```
STRICT implementation dependencies (code must exist):
  1.1 → [1.2, 1.3] → 1.4 → 1.5

Recommended batches (implementation mode):
  Batch 1: [1.1] — creates the monorepo structure
  MERGE 1.1 → dev (worktrees for batch 2 need 1.1's code)
  Batch 2: [1.2, 1.3] — can run in parallel (core vs api packages)
  MERGE 1.2, 1.3 → dev
  Batch 3: [1.4] — needs health test from 1.3 + DB from 1.2
  MERGE 1.4 → dev
  Batch 4: [1.5] — final integration
  MERGE 1.5 → dev
```

> **Critical:** Each batch requires merging to dev before the next batch starts. The worktrees for batch N+1 must be created from the updated dev branch.

**Expected output:**
- 5 branches with implemented stories
- Code review results per story
- Cross-validation report
- Shared file mutation report
- Test results per story
- Final summary with metrics

**Success criteria:**
- `npm install && npm test` passes in the CardTrader repo after all merges
- Zero TypeScript errors
- Biome lint passes

---

## Metrics to Capture

| Metric | Where | Why |
|--------|-------|-----|
| **Total wall time** per phase | Manual timer | Baseline for comparison |
| **Sub-agent dispatch count** | Step 4/3 logs | Understand overhead |
| **Corrective loops triggered** | Step 7/6 logs | Quality indicator |
| **Stories completed vs failed** | Summary report | Success rate |
| **Cross-validation issues found** | Step 8/5 report | Artifact quality indicator |
| **Shared file mutations detected** | Step 9 report | Conflict detection accuracy |
| **Context window pressure** | Observe sub-agent behavior | Identify if agents lose context |
| **Path resolution issues** | Observe during execution | Identify broken assumptions |
| **Dependency handling correctness** | Observe batch execution | Validate sequential batch logic |

---

## Pain Points to Watch For

Based on party mode discussion, these are the likely friction areas:

| # | Expected Pain Point | What to observe |
|---|---------------------|-----------------|
| 1 | **Context window saturation** | Do sub-agents lose context mid-implementation? Do they miss ACs? |
| 2 | **Path resolution** | Do story File List paths match worktree structure? |
| 3 | **Dependency management in worktrees** | Does batch 2 have batch 1's code available? |
| 4 | **Test execution claims** | Does the implementor ACTUALLY run tests or just claim success? |
| 5 | **Input resolution** | How much manual path specification does Tito need? |
| 6 | **Idle time in batches** | How much time is wasted waiting for slow sub-agents? |
| 7 | **Cross-validator scope** | Can the validator handle 5 stories with full context? |
| 8 | **Merge between batches** | Does the orchestrator handle mid-pipeline merges to dev? |

---

## What NOT to Include (Vanilla Run)

These Superpowers patterns are **excluded** from the first run:

| Pattern | Priority | Why Excluded |
|---------|----------|--------------|
| 4-Status Protocol (DONE_WITH_CONCERNS, NEEDS_CONTEXT) | P2 | Need baseline with standard 3-status first |
| "Do Not Trust The Report" | P1 | Tests reviewer behavior — measure vanilla first |
| Two-Stage Review | P3 | Structural change to step-06 — save for v2 |
| Verification-Before-Completion | P4 | Need to see if agents lie about tests first |
| Anti-Rationalization Tables | P6 | Token overhead — measure context pressure first |
| Escalation Permission | P7 | Need to see if agents soldier-on inappropriately first |

**Included (safety only):**
| Pattern | Where | Why Included |
|---------|-------|--------------|
| SUBAGENT-STOP | All sub-agent contracts | Prevents recursive orchestration — safety guard |

---

## Second Run Plan (After Vanilla Analysis)

After analyzing vanilla run results:

1. **Document all observed pain points** with specific evidence
2. **Map each pain point to potential solutions:**
   - BMO-native improvement (step file changes, contract updates)
   - Superpowers pattern adoption
   - External tool integration
   - Unsolvable limitation (Task tool, context window)
3. **Apply solutions** to step files
4. **Re-run from scratch** (fresh CardTrader repo, same artifacts)
5. **Compare metrics** against vanilla baseline
6. **Publish findings** as a BMO improvement report

---

## File Changes Required Before First Run

### Change A: Advisory Pre-Check (2 files)

**Files:**
- `_bmad/bmo/workflows/orchestrate-intellectual/steps/step-01-readiness-check.md`
- `_bmad/bmo/workflows/orchestrate-implementation/steps/step-01-readiness-check.md`

**Change:** Add new section "### 0. Pre-flight: Implementation Readiness Report Check" before "### 1. Validate Input Contract"

**Content:** See Phase 0.1 above for exact spec.

### Change B: SUBAGENT-STOP (5 files)

**Files:**
- `_bmad/bmo/workflows/orchestrate-intellectual/steps/step-03-sub-agent-dispatch.md`
- `_bmad/bmo/workflows/orchestrate-intellectual/steps/step-05-cross-validation.md`
- `_bmad/bmo/workflows/orchestrate-implementation/steps/step-04-implementor-fanout.md`
- `_bmad/bmo/workflows/orchestrate-implementation/steps/step-06-reviewer-dispatch.md`
- `_bmad/bmo/workflows/orchestrate-implementation/steps/step-08-cross-validation.md`

**Change:** Add to each sub-agent contract under "CRITICAL RESTRICTIONS":

```markdown
- SUBAGENT-STOP: You are a SUB-AGENT dispatched for a specific task.
  Do NOT invoke BMO orchestration workflows or BMAD agent menus.
  Do NOT dispatch your own sub-agents via Task tool.
  Complete YOUR assigned task and report back. Nothing else.
```

---

## References

| Document | Path | Purpose |
|----------|------|---------|
| CardTrader Brief | `docs/cardtrader-test-project-brief.md` | Product definition |
| Superpowers Research | `docs/superpowers-integration-research.md` | Patterns to adopt post-vanilla |
| PRD | `test-fixtures/planning-artifacts/prd.md` | Functional requirements |
| Architecture | `test-fixtures/planning-artifacts/architecture.md` | Technical decisions |
| Epics | `test-fixtures/planning-artifacts/epics.md` | Story breakdown |
| Stories (18) | `test-fixtures/implementation-artifacts/stories/*.md` | Implementation specs |
| BMO Config | `_bmad/bmo/config.yaml` | Orchestration settings |
| OI Workflow | `_bmad/bmo/workflows/orchestrate-intellectual/workflow.md` | Intellectual mode spec |
| OD Workflow | `_bmad/bmo/workflows/orchestrate-implementation/workflow.md` | Implementation mode spec |
| OP Workflow | `_bmad/bmo/workflows/orchestrate-pipeline/workflow.md` | Pipeline mode spec |
| Readiness Check | `_bmad/bmm/workflows/3-solutioning/check-implementation-readiness/workflow.md` | Pre-check workflow |

---

_Plan created 2026-03-10 via BMAD Party Mode planning session_
_Approved by: Rami (product owner)_
