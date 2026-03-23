# CLAUDE.md

## Overview

**v1.0.0** — A structured Software Development Lifecycle (SDLC) workflow for Claude Code, implementing a 5-phase pipeline with parallel execution support.

## Pipeline

```
Research → Plan → QA → Implement (parallel waves) → Verify
```

Each phase produces a structured artifact in `.sdlc/` and gates on human approval before proceeding.

## Agents (7)

| Agent | Phase | Model | Role |
|-------|-------|-------|------|
| `sdlc` | Orchestrator | opus | Coordinates the pipeline, dispatches phase agents, manages human gates |
| `sdlc-researcher` | 1. Research | sonnet | Explores codebase, gathers context, discovers feature gating framework |
| `sdlc-planner` | 2. Plan | opus | Maps current→proposed architecture, designs gating strategy, builds execution graph with parallel waves |
| `sdlc-qa` | 3. QA | sonnet | Shift-left: defines acceptance criteria, edge cases, and test plan BEFORE implementation |
| `sdlc-implementer` | 4. Implement | sonnet | Writes code and tests. Can be scoped to specific steps for parallel execution |
| `sdlc-verifier` | 5. Verify | opus | Reviews implementation against plan and QA criteria, delegates code review |
| `code-reviewer` | (used by verifier) | sonnet | Reviews code quality, security, performance, and correctness |

## Usage

Invoke the full pipeline:
```
Use the sdlc agent to [describe your task]
```

Or invoke individual phases:
```
Use the sdlc-researcher agent to investigate [topic]
Use the sdlc-planner agent to plan [feature]
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
