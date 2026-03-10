# Superpowers Integration Research

**Date:** 2026-03-10
**Status:** Research complete — patterns identified for adoption
**Source:** https://github.com/obra/superpowers (v5.0.0, MIT License)
**Local reference:** `integrations/superpowers/`

---

## Executive Summary

Superpowers is a composable skills framework for AI coding agents by Jesse Vincent (obra). It provides structured development workflows via markdown-based "skills" that are injected into the agent's context. **It is NOT an orchestration engine** — it delegates orchestration entirely to the AI agent's reasoning. BMO provides the orchestration infrastructure that Superpowers lacks.

**Decision: Cherry-pick patterns, NOT install as dependency.**

Reasons:
- 5 major versions in 5 months — API surface too unstable for a dependency
- 81% single-author commits (Jesse Vincent) — single-maintainer risk
- Paradigm conflict — Superpowers controls the full agent session; BMO has its own flow
- MIT license allows free adoption of any patterns

---

## Project Profile

| Metric | Value |
|--------|-------|
| Stars | 76,000 |
| Forks | 5,900 |
| Contributors | 25 (81% single author) |
| Commits | 316 |
| First release | Oct 9, 2025 |
| Latest release | v5.0.0 (Mar 9, 2026) |
| License | MIT |
| Language breakdown | Shell 66.5%, JS 19.7%, HTML 5.1%, Python 4.4%, TS 3.3% |
| Platforms | Claude Code, Cursor, Codex, OpenCode |

### Author: Jesse Vincent (obra)

- Founder/CEO of Keyboardio (ergonomic keyboards)
- Creator of Request Tracker (RT) — widely-used open source issue tracker
- 25+ years open source track record, 201 GitHub repos
- Very high credibility in the open source community

---

## Architecture Comparison

### Fundamental Difference

| Aspect | Superpowers | BMO |
|--------|-------------|-----|
| **Nature** | Prompt engineering system (markdown skills) | Orchestration engine (step file state machine) |
| **Orchestration** | AI agent interprets instructions | Explicit step files with structured transitions |
| **State** | Context window only (no persistence) | Context window but explicit tracking |
| **Runtime code** | ~208 lines JS (skill discovery only) | 33 step files, 4 workflows |
| **Sub-agents** | Sequential only (parallel PROHIBITED) | Parallel with conflict detection |
| **Recovery** | None | Full 6-step recovery workflow |
| **Conflict detection** | Avoids via serialization | Shared file mutation detection (step-09) |

### Layer-by-Layer Rating

| Layer | Superpowers | BMO |
|-------|-------------|-----|
| Prompt Engineering | 5/5 | 3/5 |
| Orchestration Engine | 1/5 (none) | 4/5 |
| State Management | 1/5 | 3/5 |
| Recovery | 1/5 (none) | 4/5 |
| Conflict Detection | 1/5 (avoidance) | 4/5 |
| Quality Gates | 5/5 | 3/5 |

**Conclusion:** The systems are complementary. BMO should adopt Superpowers' quality gate patterns while retaining its own orchestration infrastructure.

---

## Patterns to Adopt

### Priority 1: "Do Not Trust The Report" (Spec Reviewer)

**Source:** `skills/subagent-driven-development/spec-reviewer-prompt.md`

**Current BMO gap:** Step-06 (reviewer dispatch) sends a reviewer sub-agent but doesn't explicitly instruct it to distrust the implementor's claims.

**Superpowers pattern:** The spec reviewer prompt says:
> "The implementer finished suspiciously quickly. Their report may be incomplete, inaccurate, or optimistic. You MUST verify everything independently."

The reviewer is told to:
- Read actual code, NOT trust the implementor's self-report
- Check for missing requirements (things not implemented)
- Check for extra work (things implemented that weren't asked for)
- Check for misunderstandings (things implemented incorrectly)

**Adoption plan:** Update BMO's reviewer sub-agent contract (step-06 of orchestrate-implementation, step-05 of orchestrate-intellectual) to include explicit distrust language and mandate independent verification.

---

### Priority 2: 4-Status Implementor Protocol

**Source:** `skills/subagent-driven-development/SKILL.md` (lines 86-99)

**Current BMO gap:** Our implementor report has 3 statuses: `completed`, `failed`, `blocked`. Missing two important intermediate states.

**Superpowers protocol:**

| Status | Meaning | BMO Action |
|--------|---------|------------|
| `DONE` | All tasks completed successfully | Proceed to review |
| `DONE_WITH_CONCERNS` | Completed but something doesn't feel right | Review with extra scrutiny; concerns become review checklist items |
| `BLOCKED` | Cannot proceed — architectural/technical blocker | Escalate to user; don't re-dispatch blindly |
| `NEEDS_CONTEXT` | Missing information to make a decision | Provide context and re-dispatch (not a failure) |

**Key insight:** `DONE_WITH_CONCERNS` avoids unnecessary corrective loops — the implementor finished but flags uncertainty. `NEEDS_CONTEXT` avoids counting a re-dispatch as a "correction iteration" against the circuit breaker limit.

**Adoption plan:** Update the implementor contract in step-04 (orchestrate-implementation) and step-03 (orchestrate-intellectual) to use 4 statuses. Update step-05/step-04 (output collection) to handle the two new statuses appropriately.

---

### Priority 3: Two-Stage Review (Spec Compliance then Code Quality)

**Source:** `skills/subagent-driven-development/SKILL.md`, `spec-reviewer-prompt.md`, `code-quality-reviewer-prompt.md`

**Current BMO gap:** Step-06 dispatches a single reviewer that checks everything at once. This means style feedback gets mixed with correctness feedback.

**Superpowers pattern:** Two distinct review stages, executed sequentially:

1. **Spec Compliance Review** (first):
   - Does the implementation match the acceptance criteria?
   - Nothing missing? Nothing extra? Nothing misunderstood?
   - Must pass BEFORE proceeding to code quality

2. **Code Quality Review** (second, only after spec passes):
   - Architecture, patterns, naming, testing quality
   - Issue severity: Critical / Important / Minor
   - Uses git diff range (BASE_SHA..HEAD_SHA) for scope

**Why this order matters:** Catches "well-written but wrong" before spending tokens on style review. If spec doesn't pass, code quality is irrelevant — the code needs rewriting anyway.

**Adoption plan:** Split step-06 into two sub-steps (spec compliance review, then code quality review). The corrective loop (step-07) should indicate which review stage triggered the correction.

---

### Priority 4: Verification-Before-Completion Gate

**Source:** `skills/verification-before-completion/SKILL.md`

**Current BMO gap:** Step-10 (pre-merge gate) runs tests, but there's no explicit gate that forces the agent to PROVE verification happened (not just claim it).

**Superpowers pattern — The Gate Function:**

```
IDENTIFY verification command → RUN it fresh → READ full output → VERIFY claim matches output → Only THEN make claim
```

**Iron Law:** "NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE"

**Red flags (agent thinking these = STOP):**
- "I already ran the tests earlier" (stale evidence)
- "The changes are minimal so tests should still pass" (assumption)
- "Trusting agent success reports" (delegation without verification)

**Adoption plan:** Add verification evidence requirements to the implementor contract exit criteria. The implementation report must include actual test output (or hash of test output), not just `tests_passed: true`.

---

### Priority 5: SUBAGENT-STOP Gate

**Source:** `skills/using-superpowers/SKILL.md`

**Current BMO gap:** When BMO dispatches a sub-agent via Task tool, that sub-agent could theoretically trigger BMO's own orchestration skills/workflows recursively.

**Superpowers pattern:** The bootstrap skill (`using-superpowers`) includes:
> "If you were dispatched as a subagent for a specific task, SKIP this skill entirely."

This prevents recursive skill loading loops where a sub-agent dispatches its own sub-agents who load skills who dispatch more sub-agents.

**Adoption plan:** Add an explicit "You are a sub-agent. Do NOT invoke orchestration workflows or dispatch your own sub-agents" clause to all sub-agent contracts in step-04 and step-03.

---

### Priority 6: Anti-Rationalization Tables

**Source:** Multiple skills, especially `test-driven-development/SKILL.md` and `using-superpowers/SKILL.md`

**Pattern:** Skills include explicit tables of rationalizations the agent might use to skip instructions:

| Agent thinks... | Reality |
|-----------------|---------|
| "This is too simple to need a design" | Complexity surprises. Design anyway. |
| "I'll write tests after" | Tests written after are weaker. RED first. |
| "The reviewer is wrong" | Verify before dismissing. Check the actual code. |
| "I already ran the tests earlier" | Stale evidence. Run again. |

**12 specific red flag patterns** are documented as "thoughts that mean STOP."

**Adoption plan:** Add anti-rationalization tables to BMO step files where discipline is critical (review steps, pre-merge gate, corrective loop decisions).

---

### Priority 7: Escalation Permission ("When You're in Over Your Head")

**Source:** `skills/subagent-driven-development/implementer-prompt.md`

**Current BMO gap:** Our implementor contract tells the agent what to do but doesn't explicitly give permission to escalate when stuck.

**Superpowers pattern:** The implementor prompt includes a section:
> "When You're in Over Your Head — You have explicit permission to escalate. Use BLOCKED when there's a technical/architectural barrier. Use NEEDS_CONTEXT when you need information. This is not failure — it's professional judgment."

**Key insight:** Without explicit permission, agents tend to "soldier on" and produce bad code rather than admit they're stuck. The permission to escalate produces BETTER outcomes because it avoids sunk-cost-driven bad implementations.

**Adoption plan:** Add "When You're Stuck" section to BMO's implementor and document processor contracts, linking to the BLOCKED/NEEDS_CONTEXT statuses from Priority 2.

---

## Patterns NOT to Adopt

### Mandatory Brainstorming Before Everything

Superpowers forces a full brainstorm → spec → plan cycle even for simple changes. BMO should allow configurable workflow entry points. Users who already have refined stories shouldn't be forced through planning again.

### Prompt-Only Enforcement

Superpowers relies entirely on persuasive language to enforce discipline. BMO uses structural step files with explicit transitions. Our structural approach is more reliable — prompts can be rationalized away; step file transitions cannot.

### Sequential-Only Implementation

Superpowers explicitly PROHIBITS parallel implementation sub-agents (line 238 of SDD skill: "Dispatch multiple implementation subagents in parallel (conflicts)" is in the "Never" list). BMO's core value proposition is safe parallel execution with conflict detection. We solve the problem Superpowers avoids.

### Session-Start Hook Bootstrap

Superpowers injects its meta-skill into every session start. BMO is invoked on-demand via menu options, not always-on. Different activation model.

---

## Superpowers Capabilities We Already Cover Better

| Capability | Superpowers | BMO |
|------------|-------------|-----|
| Parallel sub-agent execution | Prohibited | Core feature with concurrency control |
| Worktree recovery | None | 6-step recovery workflow (orchestrate-recovery) |
| Shared file mutation detection | None (avoids via serialization) | Step-09 with snapshot + diff |
| Cross-story validation | None | Step-08 (orchestrate-implementation) |
| Corrective loop circuit breaker | No explicit limit | Configurable max_correction_loops |
| State tracking across steps | Context window only | Explicit step file state machine |
| Pipeline composition | None | orchestrate-pipeline (sequential workflow chaining) |

---

## Superpowers Capabilities Worth Monitoring

### Model Selection Guidance

Superpowers provides cost optimization guidance for sub-agent dispatch:
- **Cheap models**: Mechanical implementation (1-2 files, clear specs)
- **Standard models**: Integration and judgment tasks
- **Capable models**: Architecture, design, review

BMO doesn't currently have model selection. This could be a v2 feature if Claude Code's Task tool supports model specification.

### Pressure Testing for Skills

Superpowers tests its skills by creating pressure scenarios with combined stressors (urgency + sunk cost + authority + pragmatism). They apply 3+ pressures simultaneously to verify the agent follows instructions.

This testing methodology could be adopted for validating BMO step files and sub-agent contracts.

### Document Review Loops

Superpowers v5.0 added automated subagent review loops for specs and plans (repeat until approved or escalate after 5 iterations). This pattern is similar to BMO's corrective loop but applied to planning artifacts rather than implementation. Could be relevant for orchestrate-intellectual workflow.

---

## Reference: Superpowers File Map

Key files for future reference (all paths relative to `integrations/superpowers/`):

| File | What it contains |
|------|------------------|
| `skills/subagent-driven-development/SKILL.md` | Core SDD orchestration skill (275 lines) |
| `skills/subagent-driven-development/implementer-prompt.md` | Implementor contract template (113 lines) |
| `skills/subagent-driven-development/spec-reviewer-prompt.md` | Spec reviewer contract — "Do Not Trust" (61 lines) |
| `skills/subagent-driven-development/code-quality-reviewer-prompt.md` | Code quality reviewer contract (26 lines) |
| `skills/dispatching-parallel-agents/SKILL.md` | Parallel dispatch patterns (180 lines) |
| `skills/verification-before-completion/SKILL.md` | Verification gate function (139 lines) |
| `skills/receiving-code-review/SKILL.md` | Anti-sycophancy review handling (213 lines) |
| `skills/requesting-code-review/code-reviewer.md` | Review template with severity levels (146 lines) |
| `skills/using-superpowers/SKILL.md` | Bootstrap skill with SUBAGENT-STOP (113 lines) |
| `skills/test-driven-development/SKILL.md` | TDD enforcement with anti-rationalization (371 lines) |
| `skills/using-git-worktrees/SKILL.md` | Worktree creation patterns (218 lines) |
| `skills/finishing-a-development-branch/SKILL.md` | Branch completion with 4 options (200 lines) |
| `skills/writing-skills/SKILL.md` | Meta-skill for creating skills (655 lines) |
| `skills/writing-skills/anthropic-best-practices.md` | Official Anthropic skill authoring guide (1150 lines) |
| `agents/code-reviewer.md` | Code reviewer agent definition (48 lines) |

---

## Next Steps

1. **Adopt Priority 1-5 patterns** into BMO step files (implementation and intellectual workflows)
2. **Create test scenarios** that validate the adopted patterns work (e.g., verify spec reviewer actually distrusts implementor report)
3. **Monitor Superpowers releases** for new patterns worth adopting (especially around model selection and pipeline composition if they add it)

---

_Research conducted 2026-03-10 via BMAD Party Mode with 3 parallel investigation agents_
_Sources: Local code analysis + GitHub API + online research_
