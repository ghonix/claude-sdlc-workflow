# SDLC Workflow for Claude Code

![Version](https://img.shields.io/badge/version-v1.0.0-blue?style=flat)

A structured, agent-driven Software Development Lifecycle that brings engineering rigor to AI-assisted coding. Instead of asking Claude to "just build it," this workflow decomposes development into five disciplined phases — each with a specialized agent, a concrete artifact, and a human gate.

---

## Why This Workflow Exists

### The Problem

When AI coding assistants receive a task like "add user authentication," they tend to jump straight into writing code. This creates several failure modes:

- **Missed context** — the AI doesn't discover that the project already has a half-built auth module, or that the team uses a specific session management pattern
- **No rollback safety** — new logic ships without feature flags, making it impossible to disable without a code revert
- **Untested assumptions** — edge cases are discovered in production, not during development
- **Scope drift** — the AI adds "helpful" extras that nobody asked for, creating maintenance burden
- **Review bottleneck** — the human reviewer sees a large diff with no structure, making it hard to evaluate whether the changes are correct

### The Solution

This workflow applies the same discipline that high-performing engineering teams use — but adapted for human-AI collaboration:

1. **Understand before you act** — research the codebase and problem space before designing a solution
2. **Design before you build** — compare the current architecture against the proposed change, evaluate alternatives, and document tradeoffs
3. **Define "done" before you start** — acceptance criteria and test plans exist before the first line of code is written
4. **Build to spec** — the implementer follows the plan, not its own judgment
5. **Verify against the spec** — an independent agent checks whether what was built matches what was planned

Each phase produces a structured markdown artifact in a `.sdlc/` directory. These artifacts are human-readable, human-editable, and serve as the contract between phases. The human gates between phases give you control without requiring you to micromanage.

---

## Pipeline Overview

```
Research → [Gate] → Plan → [Gate] → QA → [Gate] → Implement → [Gate] → Verify → Done
```

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        SDLC Orchestrator (sdlc)                         │
│                    opus · coordinates all phases                         │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │
         Phase 1               │              Phase 2
    ┌─────────────┐     [Human Gate]     ┌─────────────┐
    │  RESEARCH   │─────────────────────▶│    PLAN     │
    │  sonnet     │                      │    opus     │
    │  read-only  │                      │  architect  │
    └─────────────┘                      └──────┬──────┘
    → 1-research.md                             │ [Human Gate]
                                                ▼
         Phase 3                          Phase 4
    ┌─────────────┐     [Human Gate]     ┌─────────────────────────────┐
    │     QA      │─────────────────────▶│       IMPLEMENT             │
    │   sonnet    │                      │        sonnet               │
    │  skeptical  │                      │                             │
    └─────────────┘                      │  Wave 1: [S1] ∥ [S2]       │
    → 3-qa.md                            │  Wave 2: [S3]              │
                                         │  Wave 3: [S4]              │
                                         └──────────────┬──────────────┘
                                                        │ [Human Gate]
                                                        ▼
                                                  Phase 5
                                             ┌─────────────┐
                                             │   VERIFY    │
                                             │    opus     │
                                             │   judge     │
                                             └─────────────┘
                                             → 5-verification.md
```

---

## Phase Breakdown

### Phase 1: Research

**Agent**: `sdlc-researcher` · **Model**: sonnet · **Tools**: read-only + web search

**Goal**: Understand the problem space and codebase context before anyone designs a solution.

**What it does**:

1. **Parses the task** — extracts the what, why, and constraints from the request
2. **Explores the codebase** — finds relevant files, traces data flows, identifies existing patterns and conventions
3. **Maps risks and dependencies** — surfaces migration concerns, performance-sensitive paths, and technical debt that complicates the change
4. **Discovers the feature gating framework** — searches for the project's feature flag system (LaunchDarkly, Statsig, homegrown, etc.), identifies how existing features are gated, and documents the patterns with concrete examples
5. **Surfaces prior art** — checks git history and codebase for similar past work

**Artifact**: `.sdlc/1-research.md`

```
Research Brief
├── Task (one-line summary)
├── Relevant Code (file/line table)
├── Architecture Context (data flow, dependencies)
├── Existing Patterns (conventions to follow)
├── Feature Gating Framework (library, registration, examples)
├── Risks and Concerns
├── Open Questions (for human input)
└── Suggested Scope (bigger or smaller than it sounds?)
```

**Why this phase matters**: The researcher is explicitly forbidden from proposing solutions. This separation of concerns ensures that the facts are gathered objectively — without the bias of a preferred approach coloring which parts of the codebase get explored. Every downstream phase reads this artifact.

**Key constraint**: The researcher cites specific file paths and line numbers. Vague summaries like "the auth module handles this" are rejected — the planner needs `src/auth/middleware.ts:42-78`.

---

### Phase 2: Plan

**Agent**: `sdlc-planner` · **Model**: opus · **Tools**: read-only + write

**Goal**: Design the implementation approach with explicit architecture comparison, feature gating strategy, and a parallelizable execution graph.

**What it does**:

1. **Reads and verifies the research brief** — reads the referenced source files directly, doesn't blindly trust the researcher's summaries
2. **Maps current architecture** — documents the as-is state: components, data flow, patterns, and limitations
3. **Designs proposed architecture** — describes the to-be state: what changes, what stays the same, how the new design addresses the task
4. **Evaluates alternatives** — compares the chosen approach against 1-2 alternatives with structural tradeoffs (not just vague pros/cons)
5. **Designs feature gating strategy** — determines flag name, gate type (kill switch / rollout / A/B), code placement, flag-OFF behavior, and cleanup plan
6. **Breaks the work into steps** — each step has a unique ID (S1, S2, ...), is atomic, testable, and lists exact file changes
7. **Builds an execution graph** — analyzes file dependencies and data dependencies to group steps into parallel waves

**Artifact**: `.sdlc/2-plan.md`

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
└── Dependencies
```

**Why this phase uses opus**: The planner makes the highest-stakes decisions in the workflow — architectural approach, gating strategy, and parallelism safety. These require deep reasoning about tradeoffs and second-order effects. A wrong plan means wasted implementation work; a correct plan means the implementer can execute on sonnet without needing to reason about design.

**Why current vs proposed architecture matters**: Without the explicit "before → after" comparison, plans tend to describe only the new state. This leaves the implementer guessing about what exists today and which changes are intentional vs accidental. The architecture comparison makes the delta visible and reviewable.

**The execution graph**: Steps are grouped into waves based on two rules — (1) no data dependency between steps in the same wave, and (2) no file conflicts (two steps modifying the same file must be in different waves). This enables parallel implementation without merge conflicts.

---

### Phase 3: QA (Shift-Left)

**Agent**: `sdlc-qa` · **Model**: sonnet · **Tools**: read-only

**Goal**: Define acceptance criteria, edge cases, and test plan BEFORE implementation begins.

**What it does**:

1. **Defines acceptance criteria** — for each plan step, specifies what must be true when the step is complete, what behavior should be observable, and what should NOT change
2. **Identifies edge cases** — thinks adversarially: empty inputs, null handling, boundary conditions, concurrent access, malformed data, unicode
3. **Defines the test plan** — specifies which test files need new tests, which existing tests need updating, test types (unit/integration/e2e), and critical test cases
4. **Reviews the plan for gaps** — looks for missing steps, unvalidated assumptions, unhandled error paths, and rollback risks

**Artifact**: `.sdlc/3-qa.md`

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

**Why QA runs before implementation (shift-left)**: Traditional workflows test after the code is written, which means bugs are found late and expensive to fix. By defining acceptance criteria first, two things happen:
- The **implementer** has unambiguous targets — there's no guessing about what "done" means
- The **verifier** has an objective checklist — verification becomes mechanical rather than subjective

**The QA agent is the skeptic**: The planner is optimistic by nature — it designs the happy path. The QA agent's job is to ask "what could go wrong?" It thinks about the inputs the planner didn't consider, the state transitions that are dangerous, and the regression risks that hiding in the existing test suite.

**Prioritized edge cases**: Not every edge case deserves a test. The QA agent marks each as "must test," "should test," or "nice to test" — giving the implementer a clear signal about where to spend testing effort.

---

### Phase 4: Implement

**Agent**: `sdlc-implementer` · **Model**: sonnet · **Tools**: full read/write + bash · **Permission**: acceptEdits

**Goal**: Write code and tests that follow the plan and meet the QA acceptance criteria.

**What it does**:

1. **Reads all prior artifacts** — research (context), plan (what to build), QA brief (what "done" means)
2. **Determines scope** — either implements the full plan (single-implementer mode) or specific steps only (parallel mode, e.g., "Implement S1 and S2")
3. **Implements step by step** — reads relevant files, writes code changes following existing patterns, handles edge cases from QA brief, writes specified tests, runs step verification
4. **Runs the test suite** — fixes any failures related to its changes
5. **Writes implementation summary** — documents changes made, tests added, acceptance criteria status, deviations, and known issues

**Artifact**: `.sdlc/4-implementation.md` (full plan) or `.sdlc/4-implementation-S1-S2.md` (scoped)

```
Implementation Summary
├── Changes Made (file, action, step, description)
├── Tests Added (file, count, step, status)
├── Acceptance Criteria Status (per step, checkboxes)
├── Deviations from Plan (with rationale)
└── Known Issues
```

**Why this phase uses sonnet**: If the plan is detailed enough — naming files, functions, and patterns — the implementer doesn't need deep reasoning. It needs speed and accuracy at executing a blueprint. The reasoning budget is better spent in the Plan and Verify phases where decisions are made.

**Parallel execution (waves)**: When the planner identifies independent steps, the orchestrator launches multiple implementer agents simultaneously — one per step in a wave. Each implementer is scoped to its assigned steps and forbidden from touching files belonging to other steps. This prevents merge conflicts while enabling parallel speedup.

**Feature gating enforcement**: If the plan includes a gating strategy, the implementer MUST put all new user-facing behavior behind the specified flag. The system must behave identically to today when the flag is OFF. Tests must cover both flag-ON and flag-OFF paths.

**The implementer does not commit**: Changes are left uncommitted so the verifier (and the human) can review the full diff before deciding to commit. This prevents premature commits of incomplete or incorrect work.

---

### Phase 5: Verify

**Agent**: `sdlc-verifier` · **Model**: opus · **Tools**: read-only + bash + code-reviewer delegation

**Goal**: Independently validate that the implementation is correct, complete, and ready for review.

**What it does**:

1. **Checks every acceptance criterion** — reads the relevant code to verify each criterion from the QA brief is met, marks each as PASS or FAIL with file:line evidence
2. **Delegates code review** — launches the `code-reviewer` agent to review the diff for correctness, security, performance, and adherence to existing patterns
3. **Verifies tests** — runs the test suite, confirms that tests from the QA test plan were actually written, checks edge case coverage
4. **Checks plan compliance** — compares what was built against what was planned: were all steps implemented? Were there deviations? Was anything unplanned added?
5. **Runs regression check** — verifies regression boundaries from the QA brief, confirms existing tests still pass, looks for unintended side effects

**Artifact**: `.sdlc/5-verification.md`

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

**Why this phase uses opus**: Verification is judgment-heavy. The verifier must decide whether a deviation from the plan is acceptable, whether a test adequately covers an edge case, and whether the overall implementation is sound. A false PASS is worse than a false FAIL — it lets bugs through. This requires the reasoning model's ability to evaluate nuance.

**Why the verifier delegates code review**: The `code-reviewer` agent is a specialist — it knows how to evaluate code quality, security, and performance. The verifier's job is broader: it checks the implementation against the plan, QA criteria, and regression boundaries. By delegating, neither agent duplicates the other's work.

---

## Model Allocation Strategy

| Phase | Model | Rationale |
|-------|-------|-----------|
| Research | sonnet | Fast exploration, read-heavy, no decisions |
| Plan | **opus** | Architectural decisions, tradeoff analysis, gating strategy |
| QA | sonnet | Structured output, adversarial but mechanical |
| Implement | sonnet | Follows a detailed blueprint, speed over reasoning |
| Verify | **opus** | Judgment calls on correctness, nuanced evaluation |

The principle: **reason at the boundaries, execute in the middle**. The two phases that make decisions (what to build, whether it's right) get the reasoning model. The phase that follows instructions (write this code) gets the fast model.

---

## Artifacts and Data Flow

```
.sdlc/
├── 1-research.md          ← Researcher writes, everyone reads
├── 2-plan.md               ← Planner writes, QA + Implementer + Verifier read
├── 3-qa.md                 ← QA writes, Implementer + Verifier read
├── 4-implementation.md     ← Implementer writes, Verifier reads
├── 4-implementation-S1.md  ← (parallel mode) per-step summaries
├── 4-implementation-S2.md
└── 5-verification.md       ← Verifier writes, human reads
```

Each artifact is a structured markdown file that serves as a **contract** between phases. The human can read, edit, or replace any artifact between phases — the next agent will use whatever is in the file, not what the previous agent "intended."

---

## Human Gates

The orchestrator pauses after every phase and presents a brief summary (3-5 bullet points). The human can:

- **Proceed** — move to the next phase
- **Revise** — edit the artifact and re-run the phase
- **Skip** — jump to the next phase (with a warning about consequences)
- **Abort** — stop the workflow entirely

Re-running a phase overwrites its artifact and invalidates all downstream artifacts. The orchestrator warns before proceeding.

For autonomous execution, the human can say "run it all" to skip all gates — but this is discouraged for non-trivial tasks.

---

## Parallel Execution

The planner groups implementation steps into **waves** based on two safety rules:

1. **No data dependency** — steps in the same wave don't depend on each other's output
2. **No file conflicts** — steps in the same wave don't modify the same files

```
Wave 1:  [S1] ──┐    [S2] ──┐       ← run in parallel
                 ├─→ Wave 2:  [S3] ──┐    ← sequential after Wave 1
                 │                    ├─→ Wave 3: [S4]
                 └────────────────────┘
```

The orchestrator dispatches one implementer agent per step within a wave, running them in parallel. Waves execute sequentially — Wave 2 doesn't start until all of Wave 1 completes.

If the orchestrator detects a file conflict at runtime (safety net for planner errors), it falls back to sequential execution for the conflicting steps.

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

## Scope and Limitations

**Sweet spot**: Single feature, bug fix, or focused refactor — tasks that touch 1-15 files and can be planned in 8 or fewer steps.

**Current limitations**:

| Constraint | Limit | Impact |
|---|---|---|
| Context window | ~200K tokens per agent | Large files or many files can exhaust implementer context |
| Plan step count | ~8 steps (soft cap) | More steps increase drift risk |
| Single flow | One SDLC pipeline per task | Epics with multiple independent features need separate runs |

**Future evolution**: For larger tasks, the planner could decompose the work into independent sub-tasks, each running its own full SDLC cycle (Research → Plan → QA → Implement → Verify) in parallel, with an integration phase at the end.

---

## Usage

**Full pipeline** (recommended):
```
Use the sdlc agent to [describe your task]
```

**Individual phases** (when you want finer control):
```
Use the sdlc-researcher agent to investigate [topic]
Use the sdlc-planner agent to plan [feature]
Use the sdlc-qa agent to define acceptance criteria for [plan]
Use the sdlc-implementer agent to implement [steps]
Use the sdlc-verifier agent to verify [implementation]
```

---

## Changelog

### v1.0.0

Initial release.

- 5-phase pipeline: Research → Plan → QA → Implement → Verify
- 7 agents: orchestrator + 5 phase agents + code-reviewer
- Parallel wave execution via planner's execution graph
- Shift-left QA with acceptance criteria defined before implementation
- Feature gating discovery, strategy design, and enforcement across all phases
- Human gates between every phase with skip/revise/abort options
- File-based artifact passing via `.sdlc/` directory
- Model allocation: opus for reasoning (Plan, Verify), sonnet for execution (Research, QA, Implement)
