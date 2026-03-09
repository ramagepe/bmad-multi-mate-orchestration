# BMO: BMAD Multi-Mate Orchestration — Product Brief

**Version:** 1.0
**Date:** 2026-03-09
**Status:** v1 Complete — pending integration testing
**Icon:** 🧉

_"Los flujos BMAD son los ladrillos. BMO es el albañil. Y el albañil toma mates."_

---

## 1. Problem Statement

BMAD Method workflows are sequential and single-session. When an epic has N stories, they are refined and implemented one at a time — a developer invokes a workflow, processes one story, commits, and starts over for the next.

This is inefficient. Claude Code's Task tool can invoke parallel sub-agents, and git worktrees provide filesystem isolation. BMO combines these capabilities into an orchestration layer that runs multiple stories simultaneously while maintaining safety through isolation, cross-validation, and human approval gates.

**BMO does NOT replace any existing BMAD workflow.** It composes over them — the same way a foreman doesn't lay bricks but coordinates the bricklayers.

---

## 2. What BMO Does

### Three Orchestration Modes

| Mode | Menu | What Happens | Isolation |
|------|------|-------------|-----------|
| **Intellectual** | `[OI]` | Fan-out document processing (story refinement, cross-validation, corrective loops). No code changes. | None — writes to `_bmad-output/` |
| **Implementation** | `[OD]` | Fan-out coding with git worktrees. Each sub-agent gets its own worktree/branch. Includes code review, quality gates, merge gate. | Git worktrees |
| **Pipeline** | `[OP]` | Intellectual refinement followed by implementation in a single composed flow. | Worktrees in code phase |

Plus a **Recovery** mode (`[CW]`) for cleaning up orphaned worktrees and recovering from interrupted orchestrations.

### The Golden Pattern

```
Phase 1 — SEQUENTIAL (orchestrator only):
  Validate stories → create worktrees/branches → plan batches

Phase 2 — PARALLEL (sub-agents):
  Each sub-agent works in isolation (own worktree)
  Runs existing BMM workflows (create-story, dev-story, code-review)

Phase 3 — SEQUENTIAL (orchestrator only):
  Collect outputs → cross-validate → corrective loop → human approval → push
```

### Security Model

1. **Commits**: Only in assigned worktrees, NEVER on main/base branch
2. **Push**: EXCLUSIVELY with explicit user confirmation
3. **Branches**: Orchestrator creates ALL branches BEFORE fan-out — sub-agents never create branches
4. **Shared files**: Post-hoc mutation detection — snapshot before fan-out, diff after, flag conflicts to user
5. **Circuit breaker**: Max N correction loops before escalating to human (configurable)

---

## 3. Components

### Agent

| Name | Icon | File |
|------|------|------|
| **Tito** | 🧉 | `src/agents/orchestrator.agent.yaml` |

Argentine character — cara dura, direct, humorous. Dual communication mode: colorful with the user, surgical YAML/JSON contracts with sub-agents. Research-first mentality.

### Workflows

| Workflow | Steps | Description |
|----------|-------|-------------|
| `orchestrate-intellectual` | 8 | Readiness → fan-out planning → sub-agent dispatch → output collection → cross-validation → corrective loop → human escalation → summary |
| `orchestrate-implementation` | 14 | Readiness → shared file snapshot → worktree setup → implementor fan-out → output collection → reviewer dispatch → corrective loop → cross-validation → mutation check → pre-merge gate → human approval → push → cleanup → summary |
| `orchestrate-pipeline` | 5 | Pipeline readiness → intellectual phase → phase gate → implementation phase → final summary |
| `orchestrate-recovery` | 6 | Scan worktrees → scan branches → present status → user decision → execute cleanup → verify clean state |

**Total: 33 step files** across 4 workflows.

### Input/Output Contracts

**Input (all modes):**
```yaml
input:
  epic_path: string          # Path to epic file
  stories: list[string]      # List of story file paths
  mode: intellectual | implementation | mixed
  parallel_limit: int        # From config (default: 3)
  base_branch: string        # Implementation mode only
```

**Output:**
```yaml
output:
  execution_summary:
    total_stories: int
    completed: int
    failed: int
    correction_loops_total: int
  per_story_status: list
  branches_ready_for_push: list[string]
  shared_file_mutations_detected: list
  actions_pending_user_approval: list
```

---

## 4. Configuration

Set during installation via `module.yaml` prompts, stored in `_bmad/bmo/config.yaml`:

| Variable | Description | Default | Options |
|----------|-------------|---------|---------|
| `max_parallel_agents` | Concurrent sub-agents | 3 | 2, 3, 4 |
| `max_correction_loops` | Review-fix iterations before escalation | 3 | 2, 3, 5 |
| `sub_agent_timeout_minutes` | Sub-agent timeout | 30 | 15, 30, 60 |
| `worktree_base_path` | Git worktree directory | `.claude/worktrees` | Any path |

---

## 5. Dependencies

### Required
- **BMAD Core** — workflow.xml engine, Task tool, config system
- **BMM** — create-story, dev-story, code-review workflows (composed by BMO)
- **Git** — worktree support for implementation mode
- **Claude Code CLI v2.1+** — Task tool for sub-agent dispatch

### Optional Integrations
- **TEA** — test architecture and automation (documented but not wired in v1)
- **CIS** — creative intelligence suite (available for party-mode cross-validation)

---

## 6. Development History

BMO was developed in 5 sessions (2026-03-04 to 2026-03-05) following a consistent cycle:

```
Wendy (Workflow Builder) implements → Party Mode review with 4 agents → Fixes → Commit
```

### Session Timeline

| Session | What | Commit | Lines |
|---------|------|--------|-------|
| 1 | Research spike + module brief + module creation | Initial | ~1,200 |
| 2 | orchestrate-implementation (14 steps) + review (28 fixes) | `158c02b` | +4,680 |
| 3 | orchestrate-recovery (6 steps) + review (15 fixes) | `cbf8aea` | +1,400 |
| 4 | orchestrate-intellectual (8 steps) + review (13 fixes) | `ef8a671` | +2,330 |
| 5 | orchestrate-pipeline (5 steps) + review (14 fixes) | `910db98` | +1,572 |

**Total: 70 fixes applied** across all reviews.

Originally developed inside the `instacheck` project (`_bmad/bmo/`), then extracted to this standalone repo for cross-project reuse (2026-03-09).

---

## 7. Known Limitations (v1)

| Limitation | Impact | Mitigation |
|------------|--------|------------|
| State persistence is context-window only | Long orchestrations risk context degradation | Pipeline limited to 4 stories |
| TEA integration documented but not wired | No independent test verification | Orchestrator runs tests directly |
| Task tool has no timeout enforcement | Hung sub-agents can't be killed | Timeout tracked for reporting only |
| Recovery is manual invocation only | No automatic failure hooks | User invokes [CW] manually |
| Sub-workflow return is implicit | Pipeline relies on context maintenance | Return banners in step files |
| Pipeline hard limit: 4 stories | Can't process large epics in pipeline mode | Use [OI] + [OD] separately for 5+ |

---

## 8. v2 Backlog (18 Items)

| ID | Enhancement | Priority |
|----|-------------|----------|
| v2-1 | State persistence to disk | High |
| v2-2 | TEA module wiring | High |
| v2-3 | Context window validation | High |
| v2-4 | Token cost metrics | Medium |
| v2-5 | Parallel vs sequential heuristic | Medium |
| v2-6 | BMO branch naming prefix | Low |
| v2-7 | Automated recovery from failures | Medium |
| v2-8 | Resume handoff formal contract | Medium |
| v2-9 | Audit trail to disk | Medium |
| v2-10 | Shared cleanup template | Low |
| v2-11 | Run-id namespacing | Low |
| v2-12 | Sub-agent write-path enforcement | Medium |
| v2-13 | Abort cleanup notification | Low |
| v2-14 | File-based output contracts | High |
| v2-15 | Split invoke/capture steps | Medium |
| v2-16 | Skip readiness flag from parent | Low |
| v2-17 | Implementation re-run at pipeline | Low |
| v2-18 | Per-story correction count | Low |

---

## 9. What's Next

### Immediate: Integration Testing

BMO needs to be tested end-to-end with real BMAD artifacts. The testing plan:

1. **Create test fixtures** — a small fictitious project with PRD, architecture doc, epic, and 3 stories designed to exercise specific BMO scenarios:
   - Story with independent files (happy path)
   - Story with shared files (mutation detection test)
   - Story with ambiguous AC (corrective loop test)

2. **Test [OI] first** — intellectual mode is safest (no git changes). Validates: sub-agent dispatch via Task tool, output collection, cross-validation, corrective loop.

3. **Test [OD] second** — implementation mode with the same stories. Validates: worktree creation, code implementation, review cycle, merge gate.

4. **Test [OP] last** — pipeline mode combining both. Validates: phase transitions, context maintenance across 27 steps.

### Future: Official BMAD Module

Once validated, BMO can be registered as an official module in `bmad-method` core so it appears in the installer menu alongside bmm, bmb, cis, and tea.
