---
description: Run the full ADLC pipeline (Research → Plan → QA → Implement → Verify) with parallel sub-agents
argument-hint: Task or feature description
allowed-tools: Read, Write, Glob, Grep, Bash, AskUserQuestion
---

# ADLC Pipeline (Parallel)

Coordinate the full agentic development lifecycle. You are the conductor — spawn the right agents (often in parallel) and gate each phase on human approval. Do NOT do the phase work yourself.

Task: $ARGUMENTS

## Workspace Configuration

**Before anything else**, resolve the workspace directory where all ADLC project artifacts will be stored.

Run:
```bash
cat ~/.config/adlc/config 2>/dev/null
```

- If the file exists and contains `workspace_dir`, parse and use that value as `<workspace_dir>`.
- If the file does not exist (first run ever), use `AskUserQuestion` to ask:

  > "Where should ADLC store project artifacts? This directory will hold all your ADLC workspaces (one sub-folder per task). The choice is remembered permanently across all Claude sessions and instances."

  Offer these options:
  - `~/.adlc` — hidden folder in home directory (good default for personal machines)
  - `.adlc` in the current project — project-local, checked into `.gitignore`
  - Custom path — let the user type their own

  After the user answers, expand `~` to the real absolute path (run `echo ~` to get it), then write the config:
  ```bash
  mkdir -p ~/.config/adlc
  printf '{"workspace_dir": "%s"}\n' "<chosen_absolute_path>" > ~/.config/adlc/config
  ```

  Confirm to the user: "ADLC workspace set to `<workspace_dir>`. This is saved in `~/.config/adlc/config` and will be used for all future runs."

Call the resolved absolute path `<workspace_dir>` for the rest of this run.

## Depth

Check if the user's task description includes a depth hint: `quick`, `standard`, or `thorough`. Extract and strip it. If absent, default to **standard**.

Examples:
- "Add rate limiting to the API" → no hint → `standard`
- "quick: fix the typo in the error message" → `quick`
- "thorough: redesign the authentication module" → `thorough`

Pass `Depth: <level>` to every sub-agent prompt.

## Project Directory

Derive a kebab-case slug from the task (e.g., "Add rate limiting" → `add-rate-limiting`). The project directory is `<workspace_dir>/<slug>`. Create it: `mkdir -p <workspace_dir>/<slug>`. All artifacts go here.

## Triviality Gate (Quick Path)

Before Phase 1, decide if the task qualifies for the **quick path**:
- Depth is `quick`, OR
- Task description matches trivial patterns (typo fix, single string change, comment edit, single-line config)

If quick path applies, run a **collapsed pipeline**:
1. Spawn researcher with `Scope: code` only (skip patterns, history, synthesis)
2. Skip Plan phase entirely — write a 1-step minimal plan template
3. Skip QA phase entirely — implementer reads research + makes change directly
4. Spawn implementer with `Mode: both, Depth: quick`. Add to the prompt: "No QA phase was run — implement based on the plan only. If `3-qa.md` is absent, skip QA artifact reading."
5. Spawn verifier with `Mode: criteria-check, test-run` only (parallel) + `synthesize`

Skip ALL human gates in the quick path unless verification fails. Surface only the final verdict.

Otherwise, proceed to the **standard pipeline** below.

## Standard Pipeline

```
Research (3 parallel scopes + synthesize)
  → [Gate]
  → Plan (architect → breakdown, +critique if thorough)
  → [Gate]
  → QA (criteria || adversary → synthesize)
  → [Gate]
  → Implement (parallel waves, coder || tester per step, compact briefs)
  → [Gate]
  → Verify (4 parallel evidence agents → synthesize)
```

## Phase 1: Research (Parallel Sub-Researchers)

Spawn 3 researchers in parallel using a single message with multiple Agent calls:

```
Agent(subagent_type="adlc-researcher", description="Research: code scope",
  prompt="Project directory: <workspace_dir>/<slug>\nScope: code\nDepth: <depth>\n\nTask: <task>",
  run_in_background=true)

Agent(subagent_type="adlc-researcher", description="Research: patterns scope",
  prompt="Project directory: <workspace_dir>/<slug>\nScope: patterns\nDepth: <depth>\n\nTask: <task>",
  run_in_background=true)

Agent(subagent_type="adlc-researcher", description="Research: history scope",
  prompt="Project directory: <workspace_dir>/<slug>\nScope: history\nDepth: <depth>\n\nTask: <task>",
  run_in_background=true)
```

When all three complete, spawn a synthesizer:
```
Agent(subagent_type="adlc-researcher", description="Synthesize research",
  prompt="Project directory: <workspace_dir>/<slug>\nScope: synthesize\nDepth: <depth>\n\nTask: <task>")
```

The synthesizer produces `1-research.md` (full) + `1-research-brief.md` (compact).

Present a brief summary to the user (key files, top risks, open questions).

**Gate**: "Research is complete. Review `<workspace_dir>/<slug>/1-research.md`. Proceed to planning?"

## Phase 2: Plan (Architect → Breakdown)

### Step 2a: Architect (opus)

```
Agent(subagent_type="adlc-planner", description="Architect approach",
  prompt="Project directory: <workspace_dir>/<slug>\nMode: architect\nDepth: <depth>\n\nRead 1-research-brief.md. Task: <task>")
```

Produces `2-architecture.md`.

### Step 2b: Critique (opus, optional)

If `Depth: thorough`, spawn critic in parallel with breakdown:
```
Agent(subagent_type="adlc-planner", description="Critique architecture",
  prompt="Project directory: <workspace_dir>/<slug>\nMode: critique\nDepth: <depth>\n\nRead 2-architecture.md. Task: <task>",
  run_in_background=true)
```

Produces `2-architecture-critique.md`. The breakdown agent should read it.

### Step 2c: Breakdown (sonnet)

```
Agent(subagent_type="adlc-planner", model="sonnet", description="Plan breakdown",
  prompt="Project directory: <workspace_dir>/<slug>\nMode: breakdown\nDepth: <depth>\n\nRead 2-architecture.md (and 2-architecture-critique.md if present). Task: <task>")
```

Produces `2-plan.md` + `2-plan-brief.md` + per-step `2-plan-S<N>.md` files.

Present summary (architecture summary, step count, wave count, key risks).

**Gate**: "Plan is ready. Review `<workspace_dir>/<slug>/2-plan.md`. Proceed to QA?"

## Phase 3: QA (Parallel Criteria + Adversary → Synthesize)

Spawn criteria + adversary in parallel:
```
Agent(subagent_type="adlc-qa", description="QA criteria",
  prompt="Project directory: <workspace_dir>/<slug>\nMode: criteria\nDepth: <depth>\n\nRead 2-plan-brief.md. Task: <task>",
  run_in_background=true)

Agent(subagent_type="adlc-qa", model="opus", description="QA adversary",
  prompt="Project directory: <workspace_dir>/<slug>\nMode: adversary\nDepth: <depth>\n\nRead 1-research-brief.md and 2-plan-brief.md. Task: <task>",
  run_in_background=true)
```

When both complete:
```
Agent(subagent_type="adlc-qa", description="QA synthesize",
  prompt="Project directory: <workspace_dir>/<slug>\nMode: synthesize\nDepth: <depth>\n\nMerge 3-qa-criteria.md and 3-qa-adversary.md.")
```

Produces `3-qa.md` + `3-qa-brief.md` + per-step `3-qa-S<N>.md` files.

Present summary (criteria count, top edge cases, plan gaps).

**Gate**: "QA brief is ready. Review `<workspace_dir>/<slug>/3-qa.md`. Proceed to implementation?"

## Phase 4: Implement (Parallel Waves + Coder/Tester Split)

Read the Execution Graph from `2-plan-brief.md`.

### Per-step spawning

For each step in the current wave, choose between:

**Combined mode** (default, simpler): one implementer per step with `Mode: both`.
```
Agent(subagent_type="adlc-implementer", description="Implement S1",
  prompt="Project directory: <workspace_dir>/<slug>\nMode: both\nDepth: <depth>\n\nImplement step S1. Task: <task>",
  run_in_background=true)
```

**Split mode** (faster for non-trivial steps): spawn coder + tester for the same step in parallel. Because they run simultaneously, the tester MUST use only `2-plan-S<N>.md` for the interface contract — do NOT instruct it to read in-progress implementation files.
```
Agent(subagent_type="adlc-implementer", description="Code S1",
  prompt="Project directory: <workspace_dir>/<slug>\nMode: coder\nDepth: <depth>\n\nImplement step S1 code only. Task: <task>",
  run_in_background=true)

Agent(subagent_type="adlc-implementer", description="Test S1",
  prompt="Project directory: <workspace_dir>/<slug>\nMode: tester\nDepth: <depth>\n\nWrite tests for step S1 only. Task: <task>",
  run_in_background=true)
```

Use split mode when the step involves >1 file of implementation AND has >2 acceptance criteria. Otherwise use combined mode.

### Wave coordination

Execute waves sequentially. Within each wave, launch all step agents (combined or split) in a single message with multiple tool calls so they run concurrently.

**Important**: Parallel steps within a wave MUST NOT modify the same files. Coder + tester for the SAME step can run in parallel because the tester writes test files (different paths) while the coder writes implementation files.

### Inner fix loop

If a wave finishes with test failures, spawn:
```
Agent(subagent_type="adlc-implementer", description="Fix S<N>",
  prompt="Project directory: <workspace_dir>/<slug>\nMode: fixer\nDepth: <depth>\n\nTest failures for step S<N>:\n<failure output>\n\nMake minimal targeted fix. Overwrite the existing `4-implementation-S<N>.md` summary — do not create a new file.")
```

Cap at 2 fix attempts per step. If still failing, escalate by surfacing the failure in the gate message — do NOT proceed to verify.

After all waves complete, read all `4-implementation-*.md` files and present (files changed, tests added, deviations, unmet criteria).

**Gate**: "Implementation complete. Proceed to verification?"

## Phase 5: Verify (Parallel Evidence → Synthesize)

Spawn 3-4 evidence-gathering verifiers in parallel:
```
Agent(subagent_type="adlc-verifier", model="sonnet", description="Verify: criteria",
  prompt="Project directory: <workspace_dir>/<slug>\nMode: criteria-check\nDepth: <depth>\n\nTask: <task>",
  run_in_background=true)

Agent(subagent_type="adlc-verifier", model="sonnet", description="Verify: tests",
  prompt="Project directory: <workspace_dir>/<slug>\nMode: test-run\nDepth: <depth>\n\nTask: <task>",
  run_in_background=true)

Agent(subagent_type="adlc-verifier", model="sonnet", description="Verify: regression",
  prompt="Project directory: <workspace_dir>/<slug>\nMode: regression-check\nDepth: <depth>\n\nTask: <task>",
  run_in_background=true)
```

**Conditionally spawn code-review**: Skip only if `Depth: quick`. Otherwise always spawn (git diff --shortstat is unreliable as a threshold since it measures all uncommitted repo changes, not just this ADLC run):
```
Agent(subagent_type="adlc-verifier", model="sonnet", description="Verify: code review",
  prompt="Project directory: <workspace_dir>/<slug>\nMode: code-review\nDepth: <depth>\n\nTask: <task>",
  run_in_background=true)
```

When all evidence agents complete, spawn synthesizer (opus):
```
Agent(subagent_type="adlc-verifier", description="Verify: synthesize",
  prompt="Project directory: <workspace_dir>/<slug>\nMode: synthesize\nDepth: <depth>\n\nRead all 5-verify-*.md files. Task: <task>")
```

Produces `5-verification.md`.

Present final verdict (status, critical issues, test results, recommendation).

- **PASS**: "Verification passed. Ready for commit. Commit now or review first?"
- **FAIL**: "Verification found issues: [list]. Re-run implementation to fix, or handle manually?"

## Memory Priming

Before spawning each phase's agents, scan the agent's MEMORY.md for entries tagged with keywords from the task description. Inject the top 3 most relevant memory entries into the agent prompt as:
```
Relevant memories:
- <entry 1>
- <entry 2>
- <entry 3>
```

This avoids forcing the agent to re-read its full MEMORY.md and focuses its attention.

## Phase Skipping

If the user asks to skip a phase, allow it but warn:
- Skip Research: planner has no codebase context — may miss patterns
- Skip Plan: implementer freelances — results less organized
- Skip QA: verifier only checks code quality, not requirement correctness
- Skip Verify: no formal review — consider `code-reviewer` manually

## Re-running Phases

If user wants to re-run a phase, clean up downstream artifacts first, then spawn the phase agent. Stale artifacts from the prior run will otherwise be read by subsequent phases.

| Phase to re-run | Artifacts to delete before re-running |
|-----------------|---------------------------------------|
| Research | `1-research*.md` |
| Plan | `2-*.md`, `3-*.md`, `4-*.md`, `5-*.md` |
| QA | `3-*.md`, `4-*.md`, `5-*.md` |
| Implement | `4-*.md`, `5-*.md` |
| Verify | `5-*.md` |

Run `rm -f <workspace_dir>/<slug>/<pattern>` for each matching glob before re-spawning. Tell the user which artifacts are being deleted.

## Rules

1. **Always gate on human approval** — never auto-proceed between phases unless the user says "run it all" or "autonomous mode". Exception: the quick path skips gates.
2. **Spawn in parallel** — when launching multiple agents in one phase, put them in ONE message with multiple Agent tool calls so they actually run concurrently.
3. **Wait for completion** — when agents run in background, wait for all to finish before spawning synthesizers or moving to the next wave.
4. **Pass task + project dir + depth + mode** — every agent prompt MUST include all four.
5. **Use briefs downstream** — once `1-research-brief.md` exists, downstream phases read the brief, not the full artifact (unless explicitly noted).
6. **Don't do the work yourself** — invoke phase agents.
