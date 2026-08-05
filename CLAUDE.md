# CLAUDE.md

## Overview

**v1.3.0** — A structured Software Development Lifecycle (SDLC) workflow for Claude Code, implementing a 5-phase pipeline with parallel execution support.

## Pipeline

```
[PM Proposal] → Research → Plan → QA → Implement (parallel waves) → Verify
```

The PM phase is optional — use it to define **what to build and why** before the SDLC pipeline handles **how to build it**. Each phase produces a structured artifact and gates on human approval before proceeding.

## Agents (8)

| Agent | Phase | Model | Role |
|-------|-------|-------|------|
| `sdlc-pm` | 0. Proposal | opus | Product Manager: turns rough ideas into structured project proposals — problem definition, goals, metrics, stakeholders, strategic fit |
| `sdlc` | Orchestrator | opus | Coordinates the pipeline, dispatches phase agents, manages human gates |
| `sdlc-researcher` | 1. Research | sonnet | Explores codebase, gathers context, discovers feature gating framework |
| `sdlc-planner` | 2. Plan | opus | Maps current→proposed architecture, designs gating strategy, builds execution graph with parallel waves |
| `sdlc-qa` | 3. QA | sonnet | Shift-left: defines acceptance criteria, edge cases, and test plan BEFORE implementation |
| `sdlc-implementer` | 4. Implement | sonnet | Writes code and tests. Can be scoped to specific steps for parallel execution |
| `sdlc-verifier` | 5. Verify | opus | Reviews implementation against plan and QA criteria, delegates code review |
| `code-reviewer` | (used by verifier) | sonnet | Reviews code quality, security, performance, and correctness |

## Usage

Create a project proposal (PM phase, before implementation):
```
Use the sdlc-pm agent to [describe your idea or problem]
```

Or use the slash command:
```
/sdlc-pm add dark mode support
```

Invoke the full implementation pipeline:
```
Use the sdlc agent to [describe your task]
```

Or invoke individual phases:
```
Use the sdlc-researcher agent to investigate [topic]
Use the sdlc-planner agent to plan [feature]
```

---

## ADLC Plugin

The ADLC plugin is a parallel-first variant of SDLC. Each phase decomposes into specialized sub-agents that run concurrently, with tiered artifact outputs (full + compact briefs + per-step slices) to minimize downstream token cost.

### Pipeline

```
[PM Proposal] → Research (3 parallel scopes, each dispatching scouts → synthesize)
  → Plan (architect → breakdown, +critique if thorough)
  → QA (criteria || adversary → synthesize)
  → Implement (parallel waves, coder || tester per step)
  → Verify (4 parallel evidence agents → synthesize)
```

### Agents (8)

| Agent | Phase | Model | Role |
|-------|-------|-------|------|
| `adlc-pm` | 0. Proposal | opus | Product Manager: gathers requirements, asks clarifying questions, produces a structured proposal before any code is touched |
| `adlc` | Orchestrator | sonnet | Coordinates parallel sub-agents, manages wave execution and human gates |
| `adlc-researcher` | 1. Research | sonnet | Director: dispatches scouts, analyzes evidence, writes research brief |
| `adlc-scout` | 1. Research | haiku | Lightweight search agent: finds files, greps patterns, writes structured evidence to disk |
| `adlc-planner` | 2. Plan | opus/sonnet | Architect (opus) designs approach; breakdown (sonnet) produces step graph |
| `adlc-qa` | 3. QA | sonnet/opus | Criteria (sonnet) + adversary (opus) run in parallel; synthesizer merges |
| `adlc-implementer` | 4. Implement | sonnet | Parallel per-wave, reads only scoped briefs for assigned steps |
| `adlc-verifier` | 5. Verify | sonnet/opus | Evidence modes (sonnet) gather in parallel; synthesizer (opus) produces verdict |

### Usage

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

## Artifacts

Each phase writes to `.sdlc/` in the project root:

| File | Phase | Contents |
|------|-------|----------|
| `1-research.md` | Research | Relevant code, architecture context, feature gating framework, risks |
| `2-plan.md` | Plan | Current/proposed architecture, gating strategy, execution graph with waves |
| `3-qa.md` | QA | Acceptance criteria, edge cases, test plan, plan gap analysis |
| `4-implementation.md` | Implement | Changes made, tests added, criteria status, deviations |
| `5-verification.md` | Verify | Pass/fail verdict, code review summary, test results, recommendation |

For parallel execution, implementers write scoped summaries: `4-implementation-S1-S2.md`

## Key Design Decisions

- **Reasoning at the boundaries**: Plan and Verify use opus (decisions), Research/QA/Implement use sonnet (execution)
- **Shift-left QA**: Acceptance criteria are defined BEFORE implementation, not after
- **Feature gating**: Researcher discovers the project's gating framework, planner designs the strategy, implementer enforces it
- **Parallel waves**: Planner groups independent steps into waves; orchestrator dispatches parallel implementers per wave
- **File-based data passing**: Each phase writes a `.sdlc/*.md` artifact — survives context limits, human-reviewable between phases
- **Persistent memory**: Agents learn across SDLC runs via `memory: project` — codebase patterns, past decisions, and failure patterns survive between sessions

## Agent Memory

All SDLC agents have persistent memory enabled (`memory: project` scope). Memories are stored in `.claude/agent-memory/<agent-name>/MEMORY.md` in the consuming project.

Each agent remembers role-specific learnings:
- **Researcher**: codebase architecture, key locations, feature gating details
- **Planner**: architectural decisions, complexity calibration, what worked/didn't
- **QA**: common edge cases, frequently missed issues, testing patterns
- **Implementer**: build/test commands, coding conventions, common pitfalls
- **Verifier**: recurring quality issues, review patterns, pass/fail history

Manage memories with `/sdlc-memory`:
```
/sdlc-memory              # view all agent memories
/sdlc-memory view qa      # view specific agent's memory
/sdlc-memory edit planner # edit an agent's memory
/sdlc-memory clear        # clear all memories
```
