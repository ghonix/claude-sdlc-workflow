# ADLC Workflow for Claude Code

![Version](https://img.shields.io/badge/version-v2.2.0-blue?style=flat)

A structured, agent-driven Agentic Development Lifecycle that brings engineering rigor to AI-assisted coding. Instead of asking Claude to "just build it," this workflow decomposes development into five disciplined phases — each phase itself decomposed into parallel sub-agents, with a concrete artifact and a human gate.

---

## Why This Workflow Exists

### The Problem

When AI coding assistants receive a task like "add user authentication," they tend to jump straight into writing code. This creates several failure modes:

- **Missed context** — the AI doesn't discover that the project already has a half-built auth module, or that the team uses a specific session management pattern
- **No rollback safety** — new logic ships without feature flags, making it impossible to disable without a code revert
- **Untested assumptions** — edge cases are discovered in production, not during development; bug fixes are designed around a hypothesis instead of a confirmed reproduction
- **Scope drift** — the AI adds "helpful" extras that nobody asked for, creating maintenance burden
- **Review bottleneck** — the human reviewer sees a large diff with no structure, making it hard to evaluate whether the changes are correct

### The Solution

This workflow applies the same discipline that high-performing engineering teams use — but adapted for human-AI collaboration, and parallelized so multiple sub-agents work concurrently within each phase:

1. **Understand before you act** — research the codebase and problem space before designing a solution; reproduce bugs with a failing test before hypothesizing a fix
2. **Design before you build** — compare the current architecture against the proposed change, evaluate alternatives, and document tradeoffs
3. **Define "done" before you start** — acceptance criteria and test plans exist before the first line of code is written
4. **Build to spec** — the implementer follows the plan, not its own judgment
5. **Verify against the spec** — independent evidence-gathering agents check whether what was built matches what was planned

Each phase produces tiered artifacts (full + compact brief + per-step slices) in the workspace directory. These artifacts are human-readable, human-editable, and serve as the contract between phases and between parallel sub-agents. The human gates between phases give you control without requiring you to micromanage.

---

## Pipeline Overview

```
[PM Proposal] → Research → [Gate] → Plan → [Gate] → QA → [Gate] → Implement → [Gate] → Verify → Done
```

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         ADLC Orchestrator (adlc)                         │
│                    sonnet · coordinates parallel waves                    │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │
         Phase 1                                    Phase 2
    ┌──────────────────────┐              ┌──────────────────────────┐
    │      RESEARCH         │              │          PLAN             │
    │  code ∥ patterns ∥    │──[Gate]─────▶│  architect (fable)       │
    │  history → synthesize │              │  → breakdown (sonnet)     │
    │  (scouts do search)   │              │  +critique if thorough    │
    └───────────────────────┘              └────────────┬──────────────┘
    → 1-research.md + brief                             │ [Gate]
                                                        ▼
         Phase 3                                    Phase 4
    ┌──────────────────────┐              ┌──────────────────────────────┐
    │          QA            │──[Gate]────▶│         IMPLEMENT              │
    │ criteria ∥ adversary   │              │  coder ∥ tester per step       │
    │   → synthesize          │              │                                 │
    └──────────────────────┘              │  Wave 1: [S1] ∥ [S2]           │
    → 3-qa.md + brief                     │  Wave 2: [S3]                  │
                                           └──────────────┬──────────────────┘
                                                          │ [Gate]
                                                          ▼
                                                     Phase 5
                                           ┌──────────────────────────────┐
                                           │            VERIFY              │
                                           │ 4 parallel evidence agents      │
                                           │   → synthesize (fable)          │
                                           └──────────────────────────────┘
                                           → 5-verification.md
```

---

## Phase Breakdown

### Phase 0: PM Proposal (optional)

**Agent**: `adlc-pm` · **Model**: fable · **Tools**: read-only + write + `AskUserQuestion`

**Goal**: Turn a rough idea into a structured proposal before any research or code is touched.

Asks clarifying questions, then produces a proposal covering problem statement, goals, non-goals, user stories, success metrics, scope, risks, and sizing. Written to `proposals/<slug>/proposal.md`. Use `/adlc-pm` to invoke directly.

---

### Phase 1: Research

**Agent**: `adlc-researcher` (director) + `adlc-scout` (search) · **Model**: sonnet / haiku

**Goal**: Understand the problem space and codebase context before anyone designs a solution — in parallel, across independent scopes.

**What it does**:

1. **Consults knowledge sources** — project docs (CLAUDE.md, README, ADRs), declared **context sources** (see below), MCP plugins, and skills, before touching the codebase
2. **Runs 3 scoped researchers in parallel** — `code` (relevant files, architecture, data flow), `patterns` (conventions, feature gating, testing patterns), `history` (git log, prior art, TODOs) — each dispatches lightweight `adlc-scout` sub-agents to do the actual grepping/globbing, then analyzes the evidence itself
3. **Reproduces bugs before finalizing** (bug fixes only) — writes a minimal failing test that exercises the reported behavior, runs it, and records the failure output. If reproduction isn't feasible, documents why — that's signal, not failure
4. **Synthesizes** — a 4th researcher merges the three scoped outputs into a full brief + a compact brief (~30% size) that downstream phases actually read

**Why scouts**: a lightweight haiku search agent handles Glob/Grep/Read so the reasoning-capable researcher spends its turns analyzing evidence, not executing searches.

**Artifact**: `1-research.md` (full) + `1-research-brief.md` (compact)

```
Research Brief
├── Task (one-line summary)
├── Relevant Code (file/line table)
├── Architecture Context (data flow, dependencies)
├── Existing Patterns (conventions to follow)
├── Bug Reproduction (status, test file, failure output, root cause — bug fixes only)
├── Feature Gating Framework (library, registration, examples)
├── Risks and Concerns
├── Open Questions (for human input)
├── External References (docs, ADRs, wikis consulted)
└── Suggested Scope (bigger or smaller than it sounds?)
```

**Why this phase matters**: The researcher is explicitly forbidden from proposing solutions. This separation of concerns ensures that the facts are gathered objectively — without the bias of a preferred approach coloring which parts of the codebase get explored. Every downstream phase reads the compact brief.

**Key constraint**: The researcher cites specific file paths and line numbers. Vague summaries like "the auth module handles this" are rejected — the planner needs `src/auth/middleware.ts:42-78`.

---

### Phase 2: Plan (Architect → Breakdown)

**Agent**: `adlc-planner` · **Model**: fable (architect/critique) / sonnet (breakdown)

**Goal**: Design the implementation approach with explicit architecture comparison, feature gating strategy, and a parallelizable execution graph — with reasoning and mechanical work split across model tiers.

**What it does**:

1. **Architect mode (fable)** — reads the research brief and declared context sources (ERDs, PRDs, design docs), maps current vs. proposed architecture, evaluates alternatives, designs the feature gating strategy. If the research brief confirms a bug reproduction, anchors the design on making that test pass. Writes `2-architecture.md`.
2. **Critique mode (fable, `thorough` depth only)** — runs in parallel with breakdown as a devil's advocate, challenging the chosen approach.
3. **Breakdown mode (sonnet)** — reads `2-architecture.md`, breaks the work into atomic, testable, uniquely-IDed steps (S1, S2, ...) and builds the execution graph. For bug fixes, makes S1 promote the reproduction test into the permanent suite.
4. **Decompose mode (fable, hierarchical projects only)** — for tasks large enough to split into independent streams, proposes a `project.yaml` graph instead of a flat step list; each stream later runs its own full pipeline.

**Artifact**: `2-plan.md` (full) + `2-plan-brief.md` (compact) + per-step briefs (`2-plan-S1.md`, ...)

```
Implementation Plan
├── Task
├── Current Architecture (as-is: components, data flow, diagram)
├── Proposed Architecture (to-be: what changes, what stays)
├── Alternatives Considered (table: approach, architecture change, pros, cons, why not)
├── Approach (2-3 paragraphs: transition strategy and rationale)
├── Feature Gating Strategy
│   ├── Flag name, gate type, what is gated
│   ├── Gate placement, flag-OFF behavior
│   └── Cleanup plan
├── Implementation Steps (S1, S2, S3... with files, dependencies, verification)
├── Execution Graph
│   ├── Wave table (which steps run in parallel)
│   └── Visual dependency diagram
├── Risks and Mitigations
├── Out of Scope
├── Dependencies
└── External References (ADRs, docs, RFCs that informed the plan)
```

**Why architect uses fable, breakdown uses sonnet**: The planner makes its highest-stakes decisions during architecture design — approach, gating strategy, parallelism safety. Once the architecture is fixed, turning it into a step list with file assignments is mechanical and doesn't need the reasoning model.

**Why current vs proposed architecture matters**: Without the explicit "before → after" comparison, plans tend to describe only the new state. This leaves the implementer guessing about what exists today and which changes are intentional vs accidental.

**The execution graph**: Steps are grouped into waves based on two rules — (1) no data dependency between steps in the same wave, and (2) no file conflicts (two steps modifying the same file must be in different waves).

---

### Phase 3: QA (Shift-Left)

**Agent**: `adlc-qa` · **Model**: sonnet (criteria) / fable (adversary)

**Goal**: Define acceptance criteria, edge cases, and test plan BEFORE implementation begins — with mechanical extraction and adversarial reasoning running in parallel.

**What it does**:

1. **Criteria mode (sonnet)** — mechanically extracts acceptance criteria per plan step and builds the test plan (which files, what test cases) directly from the plan
2. **Adversary mode (fable)** — reads the research and plan briefs and thinks adversarially: edge cases, failure modes, concurrency/boundary issues, plan gaps, regression boundaries
3. **Synthesize** — merges both into the full QA brief + compact brief + per-step briefs implementers actually read

Reads declared **context sources** (test standards, quality checklists, PRDs) to ground acceptance criteria and edge cases in team conventions rather than guessing.

**Artifact**: `3-qa.md` (full) + `3-qa-brief.md` (compact) + per-step briefs

```
QA Brief
├── Acceptance Criteria (per step, binary pass/fail checkboxes)
├── Edge Cases (table: scenario, expected behavior, priority)
├── Test Plan
│   ├── New Tests (file, name, type, what it validates)
│   └── Existing Tests to Update (file, reason)
├── Plan Review
│   ├── Gaps Found
│   └── Assumptions to Validate
└── Regression Boundaries (behaviors that must NOT change)
```

**Why QA runs before implementation (shift-left)**: By defining acceptance criteria first, the **implementer** has unambiguous targets and the **verifier** has an objective checklist — verification becomes mechanical rather than subjective.

**The adversary is the skeptic**: The planner is optimistic by nature — it designs the happy path. The adversary mode's job is to ask "what could go wrong?", running in parallel with the mechanical criteria extraction so neither slows the other down.

---

### Phase 4: Implement (Parallel Waves)

**Agent**: `adlc-implementer` · **Model**: sonnet · **Tools**: full read/write + bash · **Permission**: acceptEdits

**Goal**: Write code and tests that follow the plan and meet the QA acceptance criteria — with coder and tester split per step, and parallel implementers per wave.

**What it does**:

1. **Reads only scoped briefs** — per-step slices (`2-plan-S<N>.md`, `3-qa-S<N>.md`) in parallel mode, or the compact briefs in single mode; full artifacts are a fallback only
2. **Coder/tester split** — for a given step, a coder writes the implementation while a tester independently writes tests against the step's interface contract (`2-plan-S<N>.md`), avoiding the race condition of testing against in-progress coder output
3. **Determines scope** — implements the full plan (single mode) or specific steps only (parallel mode), one implementer per step within a wave
4. **Reads declared context sources** — coding guidelines and architecture docs before writing any code
5. **Runs the test suite**, writes an implementation summary

**Artifact**: `4-implementation.md` (full) or scoped summaries (`4-implementation-S1-S2.md`)

```
Implementation Summary
├── Changes Made (file, action, step, description)
├── Tests Added (file, count, step, status)
├── Acceptance Criteria Status (per step, checkboxes)
├── Deviations from Plan (with rationale)
└── Known Issues
```

**Why this phase uses sonnet**: If the plan is detailed enough — naming files, functions, and patterns — the implementer doesn't need deep reasoning. It needs speed and accuracy at executing a blueprint.

**Parallel execution (waves)**: When the planner identifies independent steps, the orchestrator launches multiple implementer agents simultaneously — one per step in a wave, each forbidden from touching files belonging to other steps.

**Feature gating enforcement**: All new user-facing behavior goes behind the plan's specified flag; the system behaves identically to today when the flag is OFF.

**The implementer does not commit**: Changes are left uncommitted so the verifier (and the human) can review the full diff before deciding to commit.

---

### Phase 5: Verify (Parallel Evidence + Synthesize)

**Agent**: `adlc-verifier` · **Model**: sonnet (evidence modes) / fable (synthesize)

**Goal**: Independently validate that the implementation is correct, complete, and ready for review — by gathering evidence in parallel, then applying judgment once.

**What it does**:

1. **Four evidence modes run in parallel (sonnet)**:
   - `criteria-check` — verifies each QA acceptance criterion against the code, PASS/FAIL with file:line evidence
   - `test-run` — runs the test suite, parses pass/fail counts, checks which tests are new
   - `code-review` — delegates to the `code-reviewer` sub-agent for correctness, security, performance
   - `regression-check` — confirms regression boundaries from the QA brief weren't crossed
2. **Synthesize (fable, the only judgment-heavy mode)** — reads all four evidence files and produces the final verdict; reads declared context sources (review checklists, test standards) as additional criteria

**Artifact**: `5-verification.md`

```
Verification Report
├── Overall Status (PASS / FAIL / PASS WITH NOTES)
├── Acceptance Criteria (table: criterion, status, evidence)
├── Code Review Summary
│   ├── Critical Issues
│   └── Warnings
├── Test Results (total, passing, failing, new tests, QA coverage)
├── Plan Compliance (steps completed, deviations, unplanned additions)
├── Regression Check (existing tests, regression boundaries)
└── Recommendation (SHIP IT / FIX AND RE-VERIFY / NEEDS REWORK)
```

**Why only synthesize uses fable**: Verification's judgment call — is a deviation acceptable, does a test adequately cover an edge case, is the overall implementation sound — happens once, after all evidence is in. A false PASS is worse than a false FAIL, so that one decision gets the reasoning model; gathering the evidence it reasons over doesn't need to.

**Why the verifier delegates code review**: The `code-reviewer` agent is a specialist — it knows how to evaluate code quality, security, and performance. The verifier's job is broader: it checks the implementation against the plan, QA criteria, and regression boundaries.

---

## Model Allocation Strategy

| Phase / Mode | Model | Rationale |
|-------|-------|-----------|
| Research (all scopes) | sonnet | Fast exploration, read-heavy, no decisions |
| Research (scout) | haiku | Pure search execution, no analysis |
| Plan — architect/critique | **fable** | Architectural decisions, tradeoff analysis, gating strategy |
| Plan — breakdown | sonnet | Mechanical: turns a fixed architecture into a step list |
| QA — criteria | sonnet | Mechanical extraction from the plan |
| QA — adversary | **fable** | Adversarial reasoning about edge cases and gaps |
| Implement | sonnet | Follows a detailed blueprint, speed over reasoning |
| Verify — evidence modes | sonnet | Mechanical evidence gathering (run tests, check criteria) |
| Verify — synthesize | **fable** | Judgment calls on correctness, nuanced evaluation |

The principle: **reason at the boundaries, execute in the middle**. Decisions get the reasoning model; execution and evidence-gathering get the fast model — and within a phase, the two are split into separate modes so neither waits on the other unnecessarily.

---

## Artifacts and Data Flow

```
<workspace_dir>/<slug>/
├── 1-research-code.md, 1-research-patterns.md, 1-research-history.md   ← scoped researchers
├── 1-research.md, 1-research-brief.md                                  ← synthesized, everyone reads brief
├── 2-architecture.md, 2-architecture-critique.md                       ← planner architect/critique
├── 2-plan.md, 2-plan-brief.md, 2-plan-S1.md, 2-plan-S2.md              ← planner breakdown
├── 3-qa-criteria.md, 3-qa-adversary.md                                 ← QA parallel modes
├── 3-qa.md, 3-qa-brief.md, 3-qa-S1.md, 3-qa-S2.md                      ← QA synthesized
├── 4-implementation.md  or  4-implementation-S1-S2.md                  ← implementer(s)
├── 5-verify-criteria.md, 5-verify-tests.md, 5-verify-review.md,
│   5-verify-regression.md                                              ← verifier evidence modes
└── 5-verification.md                                                  ← verifier synthesize, human reads
```

Each artifact is a structured markdown file that serves as a **contract** between phases and between parallel sub-agents. The human can read, edit, or replace any artifact between phases — the next agent uses whatever is in the file, not what the previous agent "intended."

---

## Human Gates

The orchestrator pauses after every phase and presents a brief summary (3-5 bullet points). The human can:

- **Proceed** — move to the next phase
- **Revise** — edit the artifact and re-run the phase
- **Skip** — jump to the next phase (with a warning about consequences)
- **Abort** — stop the workflow entirely

Re-running a phase deletes downstream artifacts before re-running, to prevent stale reads.

For autonomous execution, pass `Gate-Mode: autonomous` (used automatically in hierarchical `per-level`/`root-only` gate strategies, or the quick path) — but this is discouraged for non-trivial tasks.

---

## Parallel Execution

Parallelism happens at two levels: within a phase (parallel scopes/modes), and across implementation steps (waves).

**Within a phase**: Research runs 3 scopes in parallel (code/patterns/history), each dispatching its own scouts. QA runs criteria and adversary in parallel. Verify runs 4 evidence modes in parallel. Each converges through a synthesize step.

**Across implementation steps**, the planner groups steps into **waves** based on two safety rules:

1. **No data dependency** — steps in the same wave don't depend on each other's output
2. **No file conflicts** — steps in the same wave don't modify the same files

```
Wave 1:  [S1] ──┐    [S2] ──┐       ← run in parallel
                 ├─→ Wave 2:  [S3] ──┐    ← sequential after Wave 1
                 │                    ├─→ Wave 3: [S4]
                 └────────────────────┘
```

The orchestrator dispatches one implementer (coder + tester) per step within a wave, running them in parallel. Waves execute sequentially — Wave 2 doesn't start until all of Wave 1 completes.

---

## Feature Gating

Feature gating is woven through the pipeline as a first-class concern, not an afterthought:

| Phase | Responsibility |
|-------|---------------|
| **Research** | Discovers the project's gating framework — library, registration pattern, concrete examples |
| **Plan** | Designs the gating strategy — flag name, gate type, placement, flag-OFF behavior, cleanup plan |
| **QA** | Defines acceptance criteria for both flag-ON and flag-OFF states |
| **Implement** | Enforces gating — all new user-facing behavior behind the flag, tests for both states |
| **Verify** | Validates that flag-OFF behavior is identical to the pre-change system |

If no gating framework exists in the project, the researcher explicitly states this and the planner flags it as a risk for user-facing changes.

---

## TDD-First Bug Reproduction

For bug-fix tasks, the researcher writes and runs a minimal failing test **before** finalizing the brief, rather than letting the planner design a fix around an unverified hypothesis:

1. Researcher detects a bug-fix task (keyword heuristic, or explicit `Task-Type: bug` in the prompt)
2. Writes a minimal test that reproduces the reported behavior, using the project's existing test framework and conventions
3. Runs it, confirms it fails for the hypothesized reason, and records the failure output in the brief's **Bug Reproduction** section
4. If reproduction isn't feasible (no test framework, race condition, environment-dependent), documents why — that's signal for the planner, not a failure
5. The planner treats a confirmed reproduction as ground truth for root cause and anchors the fix design on making that test pass; the first implementation step promotes the test into the permanent suite

This section is carried verbatim through the compact brief — it's never trimmed, since it's the evidence the planner anchors on.

---

## Context Sources

Teams can declare project-specific files — ERDs, PRDs, architecture docs, coding guidelines, test standards — that agents should read before starting their protocol, instead of relying on agents to guess from tool names or stumble onto them via grep.

Sources are declared in `<workspace_dir>/context.yaml`, mapped to agent roles:

```yaml
sources:
  - path: docs/architecture.md
    description: System architecture overview
    roles: [researcher, planner]
  - path: docs/erd/payments.md
    description: Level 1 ERD for payments
    roles: [planner]
  - path: docs/prd/requirements.md
    description: Product requirements
    roles: [researcher, planner, qa]
  - path: docs/test-standards.md
    description: Testing conventions
    roles: [qa, verifier]
  - path: docs/coding-guidelines.md
    description: Coding conventions
    roles: [implementer]
```

Valid roles: `researcher`, `planner`, `qa`, `implementer`, `verifier`, `all`.

On first run, the orchestrator asks whether to configure context sources. Manage them anytime with `/adlc-context` (`setup`, `add`, `remove`, `edit`, `view`, `clear`). The orchestrator injects matching sources into each agent's spawn prompt; the agent reads them as part of its own protocol — no content is pre-loaded into the orchestrator's context.

---

## Implementer Runner

By default, the implementer writes code and tests directly in the session (`local`). You can opt into delegating each implementation step to an external coding-agent MCP tool instead (`delegate`) — useful once planning and QA have fully specified the work, so the actual code-writing token cost can be shifted off this session.

- **Opt-in, asked once**: the orchestrator prompts for this on first run, alongside the workspace directory prompt, and remembers the choice in `~/.config/adlc/config`.
- **Whole-step delegation**: delegation always covers the entire step — implementation and tests together, no coder/tester split.
- **Bring your own tool**: the plugin never hardcodes a specific MCP server or tool name. You supply the exact tool names (`delegate_create_task_tool`, `delegate_get_task_tool`) and repo identifier (`delegate_repo_param`) for whatever external coding-agent MCP server you have installed, and those live only in your local config.
- **Parallel submission, serial integration**: within a wave, delegated steps are submitted and polled in parallel (cheap I/O waits). Once the wave's tasks complete, their branches are fetched and merged into the local working tree **one at a time, in step order** — merging into a shared working tree can't safely happen in parallel.
- **Failure handling**: a failed or timed-out delegated step is surfaced in the wave summary rather than silently falling back to local implementation; you decide whether to retry locally or resubmit.

To change your choice later, edit or delete the `implementer_runner` key in `~/.config/adlc/config`.

---

## Scope and Limitations

**Sweet spot**: Single feature, bug fix, or focused refactor — tasks that touch 1-15 files and can be planned in 8 or fewer steps.

**Current limitations**:

| Constraint | Limit | Impact |
|---|---|---|
| Context window | ~200K tokens per agent | Large files or many files can exhaust implementer context |
| Plan step count | ~8 steps (soft cap) | More steps increase drift risk |
| Single flow (flat mode) | One pipeline per task | Larger epics need hierarchical decomposition (see below) |

**Hierarchical projects**: For tasks large enough to split into independent workstreams, the planner's `decompose` mode proposes a `project.yaml` graph of streams with dependencies; the orchestrator recurses, running a full pipeline per stream and gating per-stream, per-level, or root-only depending on `gate_strategy`.

---

## Usage

**Start with requirements** (recommended for new features):
```
/adlc-pm add rate limiting to the API
```

**Full pipeline**:
```
/adlc Add per-user rate limiting to the REST API: 100 req/min default, configurable per tenant
```

**Individual phases** (when you want finer control):
```
Use the adlc-researcher agent to investigate [topic]
Use the adlc-planner agent to plan [feature]
Use the adlc-qa agent to define acceptance criteria for [plan]
Use the adlc-implementer agent to implement [steps]
Use the adlc-verifier agent to verify [implementation]
```

**Manage agent memories**:
```
/adlc-memory              # view all agent memories
/adlc-memory view adlc-researcher
/adlc-memory edit adlc-planner
/adlc-memory clear
```

**Configure context sources**:
```
/adlc-context             # view configured sources
/adlc-context setup       # interactive first-time setup
/adlc-context add         # add a source
/adlc-context remove      # remove a source
```

---

## Changelog

### v2.2.0

Opt-in delegated implementation runner.

- **`adlc-implementer`**: new `Runner` parameter (`local` default, `delegate`) — delegates a whole step (coder + tester together) to a user-configured external coding-agent MCP tool instead of writing code in-session
- **Orchestrator**: first-run prompt to opt into delegation and supply your own MCP tool names/repo identifier, stored in `~/.config/adlc/config`; parallel task submission per wave, serialized git integration after the wave completes
- No vendor-specific tool names are hardcoded anywhere in the plugin — you supply your own delegate MCP server/tool names via local config

### v2.1.0

Per-agent context source system.

- **`/adlc-context` command**: `setup`, `add`, `remove`, `edit`, `view`, `clear` for a `context.yaml` config mapping project files (ERDs, PRDs, architecture docs, coding guidelines) to agent roles
- **Orchestrator**: first-run prompt to configure context sources; injects matching sources into each spawned agent's prompt based on role
- **All 5 phase agents**: new "Context Sources" protocol section — read declared files before starting

### v2.0.1 (SDLC plugin removed)

The original SDLC plugin (sequential, single-agent-per-phase) has been removed in favor of ADLC exclusively. This README and CLAUDE.md now describe ADLC only. See git history for the SDLC plugin's prior documentation and implementation if needed.

### TDD-first bug reproduction

Researchers now write and run a minimal failing test to reproduce bugs before finalizing the research brief, rather than letting the planner design a fix around an unverified hypothesis. See "TDD-First Bug Reproduction" above.

### v2.0.0

Hierarchical project management.

- **`decompose` mode** on the planner: proposes a `project.yaml` graph of independent streams with dependencies, for tasks too large for a flat step list
- **Orchestrator recursion**: walks `project.yaml`, running a full pipeline per stream, gating per-stream/per-level/root-only depending on `gate_strategy`
- **Resumability**: per-stream `status` field is the sole resumability signal; crash recovery treats on-disk artifacts as more authoritative than a stale status

### v1.3.1

Switch reasoning-tier model from opus to fable.

- **Model rename**: all agents previously using `model: opus` now use `model: fable` — PM, orchestrator, planner, verifier

### v1.3.0

ADLC PM agent and implementer context scoping.

- **adlc-pm agent**: new PM phase — asks clarifying questions before any research starts, produces a structured proposal (problem statement, goals, non-goals, user stories, success metrics, scope, risks, sizing) to `proposals/<slug>/proposal.md`
- **`/adlc-pm` command**: slash command to invoke the PM agent directly
- **Implementer minimal context**: implementers now read scoped briefs by default — parallel mode reads per-step slices (`2-plan-S<N>.md`, `3-qa-S<N>.md`), single mode reads compact briefs; full artifacts kept as fallback only
- **Removed `Use compact briefs` flag**: scoped reads are now the default behavior, not an opt-in

### v1.2.0

ADLC orchestration correctness and token efficiency.

- **Model enforcement**: planner breakdown, QA adversary, and all verifier evidence modes now receive explicit `model` overrides at spawn time — "Recommended Model" annotations are enforced, not advisory
- **Verifier synthesizer scoped reads**: synthesize mode now reads only `5-verify-*.md` evidence files, not all prior phase artifacts (removes 80-150K redundant tokens per run)
- **Agent protocol/brief alignment**: planner and QA agent protocols now read compact briefs (`1-research-brief.md`, `2-plan-brief.md`) matching what the orchestrator sends
- **Tester race condition fix**: tester mode in coder/tester split now uses `2-plan-S<N>.md` as interface contract instead of reading in-progress coder output
- **Fixer file conflict fix**: fixer mode overwrites the original `4-implementation-S<N>.md` instead of creating a separate `4-implementation-fix-S<N>.md`
- **Phase re-run cleanup**: orchestrator now deletes downstream artifacts before re-running a phase (prevents stale artifact reads)
- **Code-review condition**: changed from unreliable `git diff --shortstat` threshold to depth-based (skip for `quick`, always spawn for `standard`/`thorough`)
- **Quick path QA fallback**: implementer explicitly handles missing `3-qa.md` when QA phase was skipped
- **adlc-memory path fix**: commands now correctly resolve `adlc-<agent>` to `.claude/agent-memory/adlc-adlc-<agent>/MEMORY.md`

### v1.1.0

Persistent agent memory.

- All agents gained `memory: project` — learnings persist across runs in `.claude/agent-memory/<agent-name>/MEMORY.md`
- Each agent remembers role-specific context: researcher (architecture, key locations), planner (decisions, complexity calibration), QA (edge cases, missed issues), implementer (build commands, pitfalls), verifier (recurring issues, review patterns)
- New `/adlc-memory` command to view, edit, or clear agent memories
- Researcher, QA, and verifier gained `Write` tool access for memory updates

### v1.0.0

Initial release.

- 5-phase pipeline: Research → Plan → QA → Implement → Verify
- 7 agents: orchestrator + 5 phase agents + code-reviewer
- Parallel wave execution via planner's execution graph
- Shift-left QA with acceptance criteria defined before implementation
- Feature gating discovery, strategy design, and enforcement across all phases
- Human gates between every phase with skip/revise/abort options
- File-based artifact passing via workspace directory
- Model allocation: fable for reasoning (Plan, Verify), sonnet for execution (Research, QA, Implement)
