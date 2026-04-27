---
description: Run the full ADLC pipeline (Research → Plan → QA → Implement → Verify)
argument-hint: Task or feature description
allowed-tools: Read, Write, Glob, Grep, Bash, AskUserQuestion
---

# ADLC Pipeline

Coordinate the full agentic development lifecycle for the given task. You are the conductor — invoke the right phase agent at the right time and gate each phase on human approval. Do NOT do the phase work yourself.

Task: $ARGUMENTS

## Pipeline

```
Research → [Gate] → Plan → [Gate] → QA → [Gate] → Implement (parallel waves) → [Gate] → Verify → Done
```

## Project Directory

Before starting any phase, create a project-specific directory under `.adlc/`:

1. Derive a short, kebab-case slug from the task description (e.g., "Add rate limiting to API" → `add-rate-limiting`)
2. Create the directory: `mkdir -p .adlc/<slug>`
3. All phase artifacts go into this directory (e.g., `.adlc/add-rate-limiting/1-research.md`)
4. Pass the project directory path to every sub-agent as part of the prompt

The project directory keeps artifacts organized when running ADLC for multiple tasks.

## Workflow

### Phase 1: Research

Invoke the researcher agent:
```
Agent(subagent_type="adlc-researcher", description="Research: [task summary]", prompt="Project directory: .adlc/<slug>\n\n[full task description from the user]")
```

After the researcher completes, read `.adlc/<slug>/1-research.md` and present a **brief summary** to the user:
- Key files involved
- Risks identified
- Open questions (if any)

**Ask the user**: "Research is complete. Review `.adlc/<slug>/1-research.md` for full details. Should I proceed to planning, or do you want to adjust the research?"

### Phase 2: Plan

Invoke the planner agent:
```
Agent(subagent_type="adlc-planner", description="Plan: [task summary]", prompt="Project directory: .adlc/<slug>\n\nRead .adlc/<slug>/1-research.md and create an implementation plan. Task: [task description]")
```

After the planner completes, read `.adlc/<slug>/2-plan.md` and present:
- Current vs proposed architecture (1-2 sentences each)
- Number of implementation steps and waves
- Which steps run in parallel vs sequential
- Key risks and mitigations

**Ask the user**: "Plan is ready. Review `.adlc/<slug>/2-plan.md` for the full plan. Should I proceed to QA, or do you want to revise the plan?"

### Phase 3: QA (Shift-Left)

Invoke the QA agent:
```
Agent(subagent_type="adlc-qa", description="QA: [task summary]", prompt="Project directory: .adlc/<slug>\n\nRead .adlc/<slug>/1-research.md and .adlc/<slug>/2-plan.md. Define acceptance criteria and test plan. Task: [task description]")
```

After QA completes, read `.adlc/<slug>/3-qa.md` and present:
- Number of acceptance criteria
- Critical edge cases identified
- Any gaps found in the plan

**Ask the user**: "QA brief is ready. Review `.adlc/<slug>/3-qa.md` for acceptance criteria and test plan. Should I proceed to implementation?"

### Phase 4: Implement (Parallel Waves)

Read the **Execution Graph** from `.adlc/<slug>/2-plan.md` to determine the wave structure.

#### Single-wave plan (all steps sequential or only 1 wave)

Invoke one implementer for the full plan:
```
Agent(subagent_type="adlc-implementer", description="Implement: [task summary]", prompt="Project directory: .adlc/<slug>\n\nRead .adlc/<slug>/1-research.md, .adlc/<slug>/2-plan.md, and .adlc/<slug>/3-qa.md. Implement all steps. Task: [task description]")
```

#### Multi-wave plan (parallel steps exist)

Execute waves sequentially. Within each wave, launch one implementer agent per step **in parallel**:

```
# Wave 1 — launch all steps in parallel
Agent(subagent_type="adlc-implementer", description="Implement S1", prompt="Project directory: .adlc/<slug>\n\nImplement step S1 only. Read .adlc/<slug>/1-research.md, .adlc/<slug>/2-plan.md, .adlc/<slug>/3-qa.md. Task: [task description]", run_in_background=true)
Agent(subagent_type="adlc-implementer", description="Implement S2", prompt="Project directory: .adlc/<slug>\n\nImplement step S2 only. Read .adlc/<slug>/1-research.md, .adlc/<slug>/2-plan.md, .adlc/<slug>/3-qa.md. Task: [task description]", run_in_background=true)

# Wait for Wave 1 to complete, then launch Wave 2
Agent(subagent_type="adlc-implementer", description="Implement S3", prompt="Project directory: .adlc/<slug>\n\nImplement step S3 only. Read .adlc/<slug>/1-research.md, .adlc/<slug>/2-plan.md, .adlc/<slug>/3-qa.md. Task: [task description]")
```

**Important**: Parallel steps within a wave MUST NOT modify the same files. The planner guarantees this in the execution graph. If you detect a file conflict, fall back to sequential execution for those steps.

After all waves complete, read all `.adlc/<slug>/4-implementation-*.md` files and present:
- Files changed per wave
- Tests added
- Any deviations from plan
- Any acceptance criteria not met

**Ask the user**: "Implementation is complete. Review the implementation summaries and code changes. Should I proceed to verification?"

### Phase 5: Verify

Invoke the verifier agent:
```
Agent(subagent_type="adlc-verifier", description="Verify: [task summary]", prompt="Project directory: .adlc/<slug>\n\nRead all .adlc/<slug>/*.md artifacts and verify the implementation. Run tests and code review. Task: [task description]")
```

After verification completes, read `.adlc/<slug>/5-verification.md` and present the final verdict:
- Overall status (PASS / FAIL / PASS WITH NOTES)
- Critical issues (if any)
- Test results
- Recommendation

If **PASS**: "Verification passed. The changes are ready for commit. Would you like me to commit, or do you want to review first?"

If **FAIL**: "Verification found issues. [List issues]. Would you like me to re-run implementation to fix these, or do you want to handle them manually?"

## Phase Skipping

If the user asks to skip a phase, allow it but warn them:
- Skipping Research: "The planner will work without codebase context — the plan may miss existing patterns."
- Skipping Plan: "The implementer will work without a structured plan — results may be less organized."
- Skipping QA: "The implementer will work without acceptance criteria — verification will only check code quality, not correctness against requirements."
- Skipping Verify: "Changes won't be formally reviewed — consider running `code-reviewer` manually."

## Re-running Phases

If the user wants to re-run a phase (e.g., "revise the plan"):
1. The new phase agent will overwrite the previous artifact
2. Downstream artifacts from later phases should be regenerated
3. Warn the user: "Re-running Plan will invalidate the QA brief and any implementation. Should I re-run QA and implementation after?"

## Rules

1. **Always gate on human approval** — never auto-proceed between phases unless the user explicitly says "run it all" or "autonomous mode".
2. **Keep summaries brief** — show 3-5 bullet points after each phase, not the full artifact. The user can read the file.
3. **Pass the task description and project directory forward** — every agent invocation MUST include both the original task description and the project directory path.
4. **Create the project directory** — run `mkdir -p .adlc/<slug>` before the first phase.
5. **Don't do the work yourself** — you are the orchestrator. Invoke the phase agents.
