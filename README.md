# BMO: BMAD Multi-Mate Orchestration 🧉

Compose and orchestrate existing BMAD workflows in parallel execution with cross-validation, corrective loops, and human approval gates.

_"Los flujos BMAD son los ladrillos. BMO es el albañil. Y el albañil toma mates."_

---

## Installation

BMO is a custom module for the [BMAD Method](https://github.com/bmad-code-org/bmad-method) framework.

### Prerequisites

- BMAD Method installed (`npx bmad-method install`)
- BMAD Core + BMM modules (BMO composes over BMM workflows)
- Git with worktree support
- Claude Code CLI v2.1+

### Install as Custom Module

1. Clone this repo anywhere on your machine:

```bash
git clone https://github.com/ramagepe/bmad-multi-mate-orchestration.git
```

2. Run the BMAD installer (new install or modify existing):

```bash
npx bmad-method install
```

3. When prompted *"Would you like to install a local custom module?"*, select **Yes** and enter the path to the `src/` directory inside the cloned repo:

```
/path/to/bmad-multi-mate-orchestration/src
```

4. The installer will:
   - Read `module.yaml` and prompt for BMO configuration (parallel agents, correction loops, etc.)
   - Compile the agent (`orchestrator.agent.yaml` → `orchestrator.md`)
   - Copy all workflows and step files
   - Generate IDE commands (`.claude/commands/bmad-bmo-*.md`)
   - Register BMO in the manifest

5. Invoke the orchestrator: `/bmad-agent-bmo-orchestrator`

---

## Overview

BMO adds an orchestration layer to the BMAD ecosystem. It enables composing existing BMAD workflows into parallel execution patterns with sub-agent isolation (via git worktrees), automatic cross-validation, corrective loops with circuit breakers, and human approval gates.

Current BMAD workflows are sequential and single-session. When an epic has N stories, they are refined/implemented one at a time. BMO solves this by running multiple stories in parallel while maintaining safety through worktree isolation and post-hoc shared file mutation detection.

---

## Quick Start

1. Invoke the orchestrator agent: `/bmad-agent-bmo-orchestrator`
2. Select an orchestration mode from the menu
3. Provide your epic and story paths
4. The orchestrator handles the rest — parallel dispatch, collection, validation
5. Approve or reject merge candidates when prompted

---

## Components

### Agent

| Agent | Name | Icon | Role |
|-------|------|------|------|
| Orchestrator | Tito | 🧉 | Multi-Agent Orchestration Specialist |

### Workflows

| Workflow | Steps | Description |
|----------|-------|-------------|
| `orchestrate-intellectual` | 8 | Fan-out document processing + cross-validation + corrective loop |
| `orchestrate-implementation` | 14 | Fan-out with git worktrees + code review + quality gates + merge gate |
| `orchestrate-pipeline` | 5 | Composition of intellectual + implementation in a single flow |
| `orchestrate-recovery` | 6 | Cleanup, recovery from failures, orphaned worktree management |

---

## Modes of Operation

| Mode | Trigger | Isolation | Example |
|------|---------|-----------|---------|
| **Intellectual** | `[OI]` | No worktrees | Refine stories, validate coherence |
| **Implementation** | `[OD]` | Git worktrees | Parallel dev + review + merge gate |
| **Mixed** | `[OP]` | Worktrees in code phase | Refine → implement → validate |
| **Recovery** | `[CW]` | N/A | Cleanup worktrees, recover state |

---

## Configuration

These options are configured during installation via `module.yaml` prompts:

| Variable | Description | Default |
|----------|-------------|---------|
| `max_parallel_agents` | Maximum concurrent sub-agents | 3 |
| `max_correction_loops` | Max review-fix iterations before human escalation | 3 |
| `sub_agent_timeout_minutes` | Timeout for sub-agent execution | 30 |
| `worktree_base_path` | Base directory for git worktrees | `.claude/worktrees` |

---

## Security Model

1. **Commits**: Only in assigned worktrees, NEVER on main/base branch
2. **Push**: EXCLUSIVELY with explicit user confirmation
3. **Branch creation**: Orchestrator creates ALL branches BEFORE fan-out (sequential)
4. **Cleanup**: Worktrees cleaned up on completion or on-demand
5. **Shared file protection**: Post-hoc mutation detection with user notification

---

## Package Structure

```
src/
├── module.yaml                    # Installer contract with config prompts
├── module-help.csv                # Menu entries for BMAD help system
├── agents/
│   └── orchestrator.agent.yaml    # Tito 🧉 — source format (compiled to .md on install)
└── workflows/
    ├── orchestrate-intellectual/
    │   ├── workflow.md             # 8 step files — document refinement
    │   └── steps/
    ├── orchestrate-implementation/
    │   ├── workflow.md             # 14 step files — worktree-isolated coding
    │   └── steps/
    ├── orchestrate-pipeline/
    │   ├── workflow.md             # 5 step files — intellectual → implementation
    │   └── steps/
    └── orchestrate-recovery/
        ├── workflow.md             # 6 step files — cleanup and recovery
        └── steps/
```

---

## Dependencies

- **Requires**: BMAD Core (workflow.xml engine, Task tool), BMM (create-story, dev-story, code-review)
- **Optional integrations**: TEA (test architecture), CIS (creative intelligence)
- **External**: Git (worktree support), Claude Code CLI v2.1+

---

## Known Limitations (v1)

- **State persistence is context-window only** — no disk persistence between steps
- **TEA integration is documented but not wired** — deferred to v2
- **Pipeline mode limited to 4 stories** — use [OI] and [OD] independently for 5+
- **Task tool has no timeout enforcement** — timeout is tracked for reporting only
- **Recovery is manual invocation only** — no automatic failure hooks
- **Sub-workflow return is implicit** — relies on orchestrator context maintenance

---

## v2 Backlog

See [v2 backlog items](https://github.com/ramagepe/bmad-multi-mate-orchestration/issues) for tracked enhancements including: state persistence to disk, TEA wiring, context window validation, token cost metrics, branch naming conventions, and more.

---

## License

[MIT](LICENSE)
