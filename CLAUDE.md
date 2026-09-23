# CLAUDE.md

## Overview

**v2.2.0** — A parallel-first Agentic Development Lifecycle (ADLC) workflow for Claude Code. Each phase decomposes into specialized sub-agents that run concurrently, with tiered artifact outputs (full + compact briefs + per-step slices) to minimize downstream token cost.

## Pipeline

```
[PM Proposal] → Research (3 parallel scopes, each dispatching scouts → synthesize)
  → Plan (architect → breakdown, +critique if thorough)
  → QA (criteria || adversary → synthesize)
  → Implement (parallel waves, coder || tester per step)
  → Verify (4 parallel evidence agents → synthesize)
```

The PM phase is optional — use it to define **what to build and why** before the pipeline handles **how to build it**. Each phase produces a structured artifact and gates on human approval before proceeding.

## Agents (8)

| Agent | Phase | Model | Role |
|-------|-------|-------|------|
| `adlc-pm` | 0. Proposal | fable | Product Manager: gathers requirements, asks clarifying questions, produces a structured proposal before any code is touched |
| `adlc` | Orchestrator | sonnet | Coordinates parallel sub-agents, manages wave execution and human gates |
| `adlc-researcher` | 1. Research | sonnet | Director: dispatches scouts, analyzes evidence, writes research brief |
| `adlc-scout` | 1. Research | haiku | Lightweight search agent: finds files, greps patterns, writes structured evidence to disk |
| `adlc-planner` | 2. Plan | fable/sonnet | Architect (fable) designs approach; breakdown (sonnet) produces step graph |
| `adlc-qa` | 3. QA | sonnet/fable | Criteria (sonnet) + adversary (fable) run in parallel; synthesizer merges |
| `adlc-implementer` | 4. Implement | sonnet | Parallel per-wave, reads only scoped briefs for assigned steps |
| `adlc-verifier` | 5. Verify | sonnet/fable | Evidence modes (sonnet) gather in parallel; synthesizer (fable) produces verdict |

## Usage

Start with requirements (recommended for new features):
```
/adlc-pm add rate limiting to the API
```

Then run the full pipeline using the task description from the proposal:
```
/adlc Add per-user rate limiting to the REST API: 100 req/min default, configurable per tenant
```

Or invoke individual phases:
```
Use the adlc-researcher agent to investigate [topic]
Use the adlc-planner agent to plan [feature]
```

Manage agent memories:
```
/adlc-memory              # view all agent memories
/adlc-memory view adlc-researcher
/adlc-memory edit adlc-planner
/adlc-memory clear
```

Configure context sources (ERDs, PRDs, architecture docs, coding guidelines):
```
/adlc-context             # view configured sources
/adlc-context setup       # interactive first-time setup
/adlc-context add         # add a source
/adlc-context remove      # remove a source
```

## Artifacts

Each phase writes to the project directory (resolved from `<workspace_dir>/<slug>`):

| File | Phase | Contents |
|------|-------|----------|
| `1-research.md` | Research | Relevant code, architecture context, feature gating framework, risks |
| `2-plan.md` | Plan | Current/proposed architecture, gating strategy, execution graph with waves |
| `3-qa.md` | QA | Acceptance criteria, edge cases, test plan, plan gap analysis |
| `4-implementation.md` | Implement | Changes made, tests added, criteria status, deviations |
| `5-verification.md` | Verify | Pass/fail verdict, code review summary, test results, recommendation |

For parallel execution, implementers write scoped summaries: `4-implementation-S1-S2.md`

## Key Design Decisions

- **Reasoning at the boundaries**: Plan and Verify use fable (decisions), Research/QA/Implement use sonnet (execution)
- **Shift-left QA**: Acceptance criteria are defined BEFORE implementation, not after
- **TDD-first bug fixes**: Researcher writes a failing reproduction test before finalizing the brief; planner anchors the fix design on making that test pass
- **Feature gating**: Researcher discovers the project's gating framework, planner designs the strategy, implementer enforces it
- **Parallel waves**: Planner groups independent steps into waves; orchestrator dispatches parallel implementers per wave
- **Context sources**: Teams declare project-specific files (ERDs, PRDs, architecture docs) per agent role via `context.yaml`; agents read them before starting their protocol
- **Opt-in delegated implementation**: implementation steps can be handed off to an externally-configured coding-agent MCP tool instead of running locally, trading session tokens for async wait time
- **File-based data passing**: Each phase writes artifacts to disk — survives context limits, human-reviewable between phases
- **Persistent memory**: Agents learn across runs via `memory: project` — codebase patterns, past decisions, and failure patterns survive between sessions

## Agent Memory

All agents have persistent memory enabled (`memory: project` scope). Memories are stored in `.claude/agent-memory/<agent-name>/MEMORY.md` in the consuming project.

Each agent remembers role-specific learnings:
- **Researcher**: codebase architecture, key locations, feature gating details
- **Planner**: architectural decisions, complexity calibration, what worked/didn't
- **QA**: common edge cases, frequently missed issues, testing patterns
- **Implementer**: build/test commands, coding conventions, common pitfalls
- **Verifier**: recurring quality issues, review patterns, pass/fail history

## Context Sources

Teams can declare project-specific files that agents should read before starting their protocol. Sources are mapped to agent roles via `<workspace_dir>/context.yaml`:

```yaml
sources:
  - path: docs/architecture.md
    description: System architecture overview
    roles: [researcher, planner]
  - path: docs/erd/payments.md
    description: Level 1 ERD for payments
    roles: [planner]
  - path: docs/test-standards.md
    description: Testing conventions
    roles: [qa, verifier]
  - path: docs/coding-guidelines.md
    description: Coding conventions
    roles: [implementer]
```

Valid roles: `researcher`, `planner`, `qa`, `implementer`, `verifier`, `all`.

On first run, the orchestrator prompts for setup. Use `/adlc-context` to manage sources afterward.

## Implementer Runner

By default, the implementer writes code and tests directly in the session (`local`). You can opt into delegating each implementation step to an external coding-agent MCP tool instead (`delegate`), to conserve session tokens once planning and QA have fully specified the work.

- Opt-in only, asked once on first run alongside the workspace directory prompt; remembered in `~/.config/adlc/config`.
- Delegation always does the whole step (coder + tester together) — no split mode.
- The specific MCP server/tool names are supplied by you as local config values (`delegate_create_task_tool`, `delegate_get_task_tool`, `delegate_repo_param`) — never hardcoded in the plugin.
- Steps in a wave are submitted and polled in parallel (cheap I/O), but their resulting branches are merged into the local working tree one at a time, sequentially, after the wave completes — parallel git operations on one shared tree aren't safe.
- To change your choice later, edit or delete the `implementer_runner` key in `~/.config/adlc/config`.
