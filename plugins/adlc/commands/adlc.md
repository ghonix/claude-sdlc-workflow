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

Derive a kebab-case slug from the task (e.g., "Add rate limiting" → `add-rate-limiting`). For a top-level invocation, resolve the single variable `<project-dir>` used everywhere else in this file:

```
<project-dir> = <workspace_dir>/<slug>
```

Create it: `mkdir -p <project-dir>`. All artifacts go here.

Every reference below to a project artifact path (`Agent(...)` prompts, skeleton-file heredocs, cleanup globs) uses `<project-dir>` — never re-derive `<workspace_dir>/<slug>` inline.

## Project Mode Detection

Immediately after creating `<project-dir>`, check whether this is a hierarchical project:

```bash
test -f "<project-dir>/project.yaml" && echo "hierarchical" || echo "flat"
```

- **`flat`** (no `project.yaml` present): proceed exactly as today — go to the **Leaf Task Pipeline** below, starting at the Triviality Gate. This is the only path today; all 3 existing flat projects on disk resolve here unchanged.
- **`hierarchical`** (`project.yaml` present): skip the Triviality Gate and the Leaf Task Pipeline entirely — go straight to **Hierarchical Project Loop** below.

Note: no new `allowed-tools` entry is required for this detection or for any future `project.yaml` parsing — `Bash` (already declared in this command's frontmatter) covers `test -f` and shell/`python3 -c`-based reads.

## Hierarchical Project Loop

This section implements `process_project(<project-dir>)`. Treat it as a function: whenever a step below says "recurse", it means re-enter this entire section with `<project-dir>` rebound to the sub-stream's directory — it is an internal prose loop, NOT a new `Agent(subagent_type="adlc", ...)` spawn.

**Resumability**: each stream's per-stream `status` field in `project.yaml` is the *sole* resumability signal for this whole loop — there is no separate state file, lock file, or run-log. Before building the dependency graph on any invocation, check whether `<project-dir>/project.yaml` already exists (this was already done once by `## Project Mode Detection` to route into this loop, but restate it here as the resumability rule): if it exists, Step 0 (first-time decomposition) is skipped unconditionally and the loop proceeds straight to Step 1 (Parse + graph) — a second `/adlc <task>` invocation against a project directory that already has `project.yaml` never re-runs decomposition and never re-prompts the user to re-approve the proposed streams, even if the task description repeated verbatim looks identical to the original call. Crash recovery (Step 4 below) additionally treats on-disk phase artifacts as more authoritative than a stale `status` field, so an orchestrator that died mid-run resumes from the correct phase rather than restarting the stream from scratch.

### Step 0: First-time decomposition

If `<project-dir>/project.yaml` does not exist yet (this only happens on the very first entry into hierarchical mode, since `## Project Mode Detection` already routed us here based on its presence — this check matters for recursive re-entries, where a sub-stream directory might not have a `project.yaml` of its own yet and should instead go through the Leaf path, never here):

1. Spawn `Agent(subagent_type="adlc-planner", model="opus", description="Decompose project", prompt="Project directory: <project-dir>\nMode: decompose\nDepth: <depth>\n\nTask: <task>")`.
2. Present the proposed `project.yaml` (streams, `depends_on` graph, `gate_strategy`) to the user.
3. **Gate**: "Proposed decomposition into N streams with `gate_strategy: <value>`. Review `<project-dir>/project.yaml`. Approve, edit, or cancel?" Do NOT create any `<project-dir>/<stream-slug>/` directory and do NOT proceed to Step 1 until the user approves (editing `project.yaml` in place counts as approval of the edited version — re-read it after edits).
4. Once approved, create empty stream directories: `mkdir -p "<project-dir>/<stream-slug>"` for every entry in `streams[]`. Do not put anything else in them yet — each stream's own pipeline creates its own artifacts when its turn comes up.

### Step 1: Parse and validate `project.yaml`

At the start of **every** wave iteration (not just once) — this is a deliberate self-correcting checkpoint against prose-loop drift — re-read `<project-dir>/project.yaml` from disk.

**Orphaned `.tmp` recovery (F2)**: before reading `project.yaml` itself, check for a leftover `<project-dir>/project.yaml.tmp` — this can only exist if the atomic write in Step 4 (temp-write then `mv`) was interrupted between the two, e.g. by a crash. If present:

```bash
if [ -f "<project-dir>/project.yaml.tmp" ]; then
  python3 -c "
import yaml, sys, shutil
try:
    with open('<project-dir>/project.yaml.tmp') as f:
        data = yaml.safe_load(f)
    if isinstance(data, dict) and 'streams' in data:
        shutil.move('<project-dir>/project.yaml.tmp', '<project-dir>/project.yaml')
        print('Recovered orphaned project.yaml.tmp (interrupted atomic write) — promoted it to project.yaml.', file=sys.stderr)
    else:
        print('WARNING: project.yaml.tmp exists but did not parse to a valid streams mapping — leaving both files as-is; inspect <project-dir>/project.yaml.tmp manually.', file=sys.stderr)
except Exception as e:
    print(f'WARNING: project.yaml.tmp exists but failed to parse ({e}) — leaving both files as-is; inspect <project-dir>/project.yaml.tmp manually.', file=sys.stderr)
"
fi
```

Only promote the `.tmp` file if it parses as valid YAML AND contains a `streams` key — otherwise leave both files untouched and surface the warning rather than guessing which one is authoritative. This check runs on every wave iteration's re-read, not just once, since a crash could in principle happen between wave iterations as well as mid-status-write.

With any orphaned `.tmp` handled, parse `project.yaml`:

```bash
python3 -c "
import yaml, json, sys
try:
    with open('<project-dir>/project.yaml') as f:
        raw = f.read()
except FileNotFoundError:
    print('ERROR: project.yaml not found at <project-dir>', file=sys.stderr); sys.exit(1)

if not raw.strip():
    print('ERROR: project.yaml is empty. Delete it and re-run to re-decompose (Mode: decompose), or restore it from project.yaml.bak if present.', file=sys.stderr)
    sys.exit(1)

try:
    data = yaml.safe_load(raw)
except yaml.YAMLError as e:
    print(f'ERROR: project.yaml is not valid YAML: {e}', file=sys.stderr); sys.exit(1)

if not isinstance(data, dict):
    print('ERROR: project.yaml did not parse to a mapping. Delete it and re-run to re-decompose.', file=sys.stderr); sys.exit(1)

print(json.dumps(data))
" 2>&1
```

If this fails with `ModuleNotFoundError: No module named 'yaml'`, do not proceed and do not fall back to flat mode — report clearly: "PyYAML is required for hierarchical projects. Install it with `pip3 install pyyaml`." and stop the run entirely (this applies every time `project.yaml` is read, not just at decompose time).

**Schema validation** (run before building the dependency graph, on every read — cheap and catches hand-edits per E9):
- Required top-level fields present: `name`, `description`, `gate_strategy`, `streams`. Missing any → `ERROR: project.yaml is missing required field: <field>`.
- Each stream has `name`, `slug`, `status`, `depends_on`. Missing any → `ERROR: stream <index> is missing required field: <field>`.
- Every `slug` matches `^[a-z0-9]+(-[a-z0-9]+)*$`. Violation → `ERROR: stream slug '<slug>' is not valid kebab-case`.
- No duplicate slugs across `streams[]`. Violation → `ERROR: duplicate stream slug: '<slug>'`.
- Every entry in every stream's `depends_on` resolves to another slug present in this same `streams[]` list (sibling-only — a slug belonging to a not-yet-created sub-stream one level down is out of scope and invalid here). Violation → `ERROR: stream '<slug>' depends_on unknown or out-of-scope slug '<dep>'`.
- `gate_strategy` is one of `per-stream`, `per-level`, `root-only`. If missing or invalid: do not abort — fall back to the same smart default the planner uses (`per-stream` if `len(streams) <= 3` else `per-level`) and warn the user: "`gate_strategy: '<value>'` is invalid; defaulting to `<default>`." (E8).

On any `ERROR:` above, stop processing this project level entirely and surface the error to the user — do not attempt partial execution.

### Step 2: Build the dependency graph and topologically sort into waves

Using the validated `streams[]` list, build a directed graph on `depends_on` edges (same wave-grouping approach already used for implementation steps in Phase 4's Execution Graph). Detect cycles explicitly — do not let a cyclic graph silently loop or silently skip unreachable streams:

- Run a standard Kahn's-algorithm topological sort (repeatedly peel off nodes with in-degree 0 into successive waves).
- If nodes remain after no more in-degree-0 nodes can be peeled off, those remaining nodes form one or more cycles. Report: `ERROR: circular dependency detected among streams: <slug-a> -> <slug-b> -> ... -> <slug-a>` (trace the cycle by following `depends_on` edges among the stuck nodes) and stop — do not execute any wave.

If all `streams[]` already have `status: completed`, recognize the project as fully done immediately: skip straight to Step 5 (rollup summary) without re-running or re-decomposing anything (E5).

### Step 3: Per-wave execution

For each wave (in topological order), for each stream in that wave:

1. **Skip** if `status == completed` — nothing to do.
2. **Blocked check**: if any `depends_on` entry's `status != completed`, this stream stays `blocked` this iteration. Write `status: blocked` for it in `project.yaml` if not already so (see Step 4 for the write mechanics) — this is the one case where the orchestrator, not the planner, writes a status value. Re-evaluate on the next wave iteration once dependencies are re-read as `completed`; there is no separate manual "unblock" action, `blocked` clears itself automatically the moment its `depends_on` streams complete.
3. **Determine leaf vs. decomposition point**: `test -f "<project-dir>/<stream-slug>/project.yaml"`.
   - **No** → **Leaf**: rebind `<project-dir>` to `<project-dir>/<stream-slug>` for the duration of this stream, and run the **Leaf Task Pipeline** (below) against that rebound path, with `<task>` set to this stream's `description`. Pass `Gate-Mode: gated` if `gate_strategy` is `per-stream`, otherwise pass `Gate-Mode: autonomous` (see "Gate-Mode parameter" at the top of the Leaf Task Pipeline section) — this is how `per-level`/`root-only` suppress the Leaf Task Pipeline's hardcoded per-phase gates while `per-stream` keeps them. Set the stream's `status` in the parent `project.yaml` to track pipeline progress as it moves through phases (`research` while Phase 1 runs, `plan` during Phase 2, `qa` during Phase 3, `implement` during Phase 4, `verify` during Phase 5, `completed` once Phase 5 passes) — these intermediate values give `/adlc-status` something meaningful to render mid-run; writing all of them is REQUIRED, not optional, whenever `gate_strategy` is `per-stream` (see Step 4), and MAY be batched to just the final value under `per-level`/`root-only` if intermediate writes would be discarded before the next read anyway.
   - **Yes** → **Decomposition point**: recurse — re-enter this entire `process_project` loop (Steps 0-5) with `<project-dir>` rebound to `<project-dir>/<stream-slug>` and `<task>` set to this stream's `description` field from `project.yaml`. Note (v1 scope): the orchestrator only detects and processes a sub-`project.yaml` if one has already been manually seeded in `<project-dir>/<stream-slug>/`; it does NOT auto-trigger `Mode: decompose` against a leaf stream to create one. Recursive auto-decomposition is a future enhancement, not implemented in v1. The sub-project's own root-level `status` in the *parent's* `project.yaml` is set to `completed` only once the recursive call's own Step 5 (rollup) reports the whole sub-tree done; while the recursive call is in progress, keep the parent-level entry at whatever phase-tracking value made sense before recursing in (or simply `implement` as a coarse placeholder — the sub-tree's own `project.yaml` is the authoritative detail).
4. **Stream failure**: if the leaf pipeline's verifier reports FAIL after the fix-loop cap is exhausted, or a phase agent errors/times out/produces corrupt output, set that stream's `status` to `failed` (a value distinct from `blocked` and `completed`). A `failed` stream:
   - Does NOT halt sibling streams already running or queued in the same wave — they continue independently.
   - DOES permanently block any other stream whose `depends_on` includes the failed slug (they remain `blocked` indefinitely; this must be called out explicitly in the Step 5 rollup as needing human intervention, not silently retried).
   - Is never auto-retried by the loop; the user must either fix it manually and flip `status` back (e.g., to `pending`) in `project.yaml`, or re-run that stream's phase explicitly (same "Re-running Phases" mechanics as the flat pipeline, scoped to that stream's directory).

### Step 4: Update status (atomic write)

After a stream's Leaf Task Pipeline phase-transition or full recursive sub-tree finishes (success, `blocked`, or `failed`), update its `status` field in the **parent** `project.yaml` — never the sub-stream's own directory — using an atomic write so a crash or a concurrent hand-edit (E9) never leaves a partially-written file:

```bash
python3 -c "
import yaml
with open('<project-dir>/project.yaml') as f:
    data = yaml.safe_load(f)
for s in data['streams']:
    if s['slug'] == '<stream-slug>':
        s['status'] = '<new-status>'
with open('<project-dir>/project.yaml.tmp', 'w') as f:
    yaml.safe_dump(data, f, sort_keys=False)
"
mv "<project-dir>/project.yaml.tmp" "<project-dir>/project.yaml"
```

**Crash recovery (E10)**: if the orchestrator is interrupted after a stream's pipeline completed but before this status write lands, on-disk artifacts are authoritative. On the next run, before re-running a stream whose status still shows an in-progress value (e.g. `verify`), check whether `<stream-dir>/5-verification.md` already exists and reports PASS — if so, just write `status: completed` and move on rather than re-running the whole stream. Apply the same "trust the artifact over the status field" logic per-phase (e.g. a `4-implementation-*.md` already present means Phase 4 doesn't need to re-run even if `status` still reads `implement`).

**Orphaned sub-project check (E11)**: if a stream is a decomposition point (has its own `project.yaml`) but that file's `streams[]` is empty or all entries fail validation, do not silently mark it `completed` — report: `ERROR: '<stream-slug>' has a project.yaml with no valid streams; inspect it manually` and leave its status unchanged.

### Step 5: Gate strategy dispatch

Read `gate_strategy` (validated in Step 1) and apply it uniformly for the duration of processing this project level (a recursive call may use a different `gate_strategy` from its parent — each `project.yaml` owns its own value):

- **`per-stream`**: no change from the flat pipeline's behavior — dispatch every leaf stream's Leaf Task Pipeline with `Gate-Mode: gated`, so every phase transition inside it gates exactly as it does today (Research gate, Plan gate, QA gate, Implement gate, Verify gate), for every stream, independently.
- **`per-level`**: dispatch every leaf stream's Leaf Task Pipeline with `Gate-Mode: autonomous` — this suppresses all five per-phase gates inside individual leaf streams (see "Gate-Mode parameter" in the Leaf Task Pipeline section). Instead, the orchestrator itself gates once per phase per wave: run the same phase (e.g. Phase 1: Research) across **every stream in the current wave** before gating once. Semantics for heterogeneous timing within a wave: **wait-for-slowest** — if streams in a wave are running the same phase in parallel (per the "Spawn in parallel" rule) but finish at different times, hold the gate until all streams in the wave have completed that phase; do not batch-fire the gate as soon as the first stream finishes, and do not let a fast stream silently start its next phase early. This keeps the gate meaningful as a single checkpoint over the whole wave rather than a race.
- **`root-only`**: dispatch every leaf stream's Leaf Task Pipeline with `Gate-Mode: autonomous` (same suppression as `per-level`) and additionally suppress the orchestrator's own per-wave-per-phase gate from `per-level` — no per-stream, no per-level, no gates inside recursive sub-projects at any depth. The only gate for the whole tree is Step 0's initial decomposition-approval gate, plus one final gate after Step 5's rollup below: "All streams completed. Review the rollup. Proceed?"

### Step 6: Completion and rollup

Once every stream in `streams[]` (after re-reading `project.yaml`) has `status == completed`:

1. Mark this project level done (no separate field needed — "all streams completed" is the done condition itself).
2. Present a rollup summary: one line per stream, reusing each leaf stream's `4-implementation-*.md`/`5-verification.md` summaries (files changed, verdict) and, for decomposition-point streams, the recursive sub-tree's own rollup line. Call out any `failed` or still-`blocked` streams prominently even if the immediate wave is otherwise done — a project is only fully complete when zero streams are `failed` or `blocked`.
3. If this is the root-level call and `gate_strategy: root-only`, present the final gate described in Step 5 before returning control to the user.

### Hierarchical Gotchas

- **`project.yaml` has exactly one writer**: the orchestrator (this loop, via Step 4's atomic write) is the sole writer of `project.yaml` for a given project level, at every point after the initial decomposition gate in Step 0. Never hand-edit `project.yaml` while the orchestrator is mid-run (E9) — a concurrent hand-edit racing the orchestrator's read-modify-atomic-write cycle in Step 4 can silently clobber your edit (last writer wins) or be clobbered by it. The one supported exception is Step 0's approval gate itself, where editing the freshly-proposed `project.yaml` in place *is* the documented way to approve an edited version, precisely because the orchestrator hasn't started writing to it yet.
- If you need to change a stream's status by hand (e.g. to un-stick a `failed` stream per Step 3), do it only while the orchestrator is not running, and prefer the documented re-run mechanics below over freehand edits where possible.
- The atomic write (`project.yaml.tmp` → `mv`) means readers should never observe a torn/partial `project.yaml`; if you ever see one, it indicates something bypassed the documented write path.

## Leaf Task Pipeline

Everything from **Triviality Gate** through **Phase 5: Verify** below is the Leaf Task Pipeline — it runs a single, flat, non-hierarchical task end-to-end. It is used both for ordinary flat invocations (`flat` mode above) and, once implemented, for each individual leaf stream inside a hierarchical project.

**Gate-Mode parameter**: this pipeline accepts an implicit `Gate-Mode: autonomous | gated` setting (default `gated`, matching today's behavior for flat invocations, where the parameter is simply never set to `autonomous`).
- `Gate-Mode: gated` (default): every `**Gate**:` prompt below runs exactly as written — pause and wait for human approval before proceeding.
- `Gate-Mode: autonomous`: skip every `**Gate**:` prompt below (the 5 phase gates: Research, Plan, QA, Implement, Verify) and proceed straight through to the next phase without pausing, presenting the same summaries but not waiting for a response. This is set by the Hierarchical Project Loop (Step 3's leaf branch) when `gate_strategy` is `per-level` or `root-only`, so the orchestrator can own cross-stream gating itself (Step 5) instead of the pipeline gating per-phase per-stream. The Triviality Gate's quick-path gate-skipping behavior is independent of this parameter and unaffected by it.

### Triviality Gate (Quick Path)

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

### Standard Pipeline

```
[PM Proposal] → Research (3 parallel scopes + synthesize)
  → [Gate]
  → Plan (architect → breakdown, +critique if thorough)
  → [Gate]
  → QA (criteria || adversary → synthesize)
  → [Gate]
  → Implement (parallel waves, coder || tester per step, compact briefs)
  → [Gate]
  → Verify (4 parallel evidence agents → synthesize)
```

### Phase 1: Research (Parallel Sub-Researchers)

**Before spawning researchers**, create skeleton output files so researchers only need to `Edit` (not `Write`). This prevents the common failure where a researcher returns findings as chat text without writing to disk.

```bash
for scope in code patterns history; do
cat > "<project-dir>/1-research-${scope}.md" << 'SKELETON'
# Research Brief — ${scope} Scope

## Task
[Task description will be filled by researcher]

## Relevant Code
_Research in progress..._

## Architecture Context
_Research in progress..._

## Existing Patterns
_Research in progress..._

## Risks and Concerns
_Research in progress..._

## Open Questions
_Research in progress..._
SKELETON
done
```

Then spawn 3 researchers in parallel using a single message with multiple Agent calls:

```
Agent(subagent_type="adlc-researcher", description="Research: code scope",
  prompt="Project directory: <project-dir>\nScope: code\nDepth: <depth>\n\nTask: <task>\n\nIMPORTANT: Your output file <project-dir>/1-research-code.md already exists on disk as a skeleton. Use the Edit tool to replace each '_Research in progress..._' section with your findings. Do NOT return findings as chat text — Edit the file on disk.",
  run_in_background=true)

Agent(subagent_type="adlc-researcher", description="Research: patterns scope",
  prompt="Project directory: <project-dir>\nScope: patterns\nDepth: <depth>\n\nTask: <task>\n\nIMPORTANT: Your output file <project-dir>/1-research-patterns.md already exists on disk as a skeleton. Use the Edit tool to replace each '_Research in progress..._' section with your findings. Do NOT return findings as chat text — Edit the file on disk.",
  run_in_background=true)

Agent(subagent_type="adlc-researcher", description="Research: history scope",
  prompt="Project directory: <project-dir>\nScope: history\nDepth: <depth>\n\nTask: <task>\n\nIMPORTANT: Your output file <project-dir>/1-research-history.md already exists on disk as a skeleton. Use the Edit tool to replace each '_Research in progress..._' section with your findings. Do NOT return findings as chat text — Edit the file on disk.",
  run_in_background=true)
```

When all three complete, **verify output files exist and are not still skeletons**:
```bash
for scope in code patterns history; do
  f="<project-dir>/1-research-${scope}.md"
  if grep -q "Research in progress" "$f" 2>/dev/null; then
    echo "WARNING: $f still contains skeleton placeholders"
  elif [ ! -s "$f" ]; then
    echo "WARNING: $f is empty or missing"
  else
    echo "OK: $f"
  fi
done
```

If any researcher failed to write, log a warning and continue — the synthesizer will work with whatever is available.

Then spawn a synthesizer:
```
Agent(subagent_type="adlc-researcher", description="Synthesize research",
  prompt="Project directory: <project-dir>\nScope: synthesize\nDepth: <depth>\n\nTask: <task>")
```

The synthesizer produces `1-research.md` (full) + `1-research-brief.md` (compact).

Present a brief summary to the user (key files, top risks, open questions).

**Gate**: "Research is complete. Review `<project-dir>/1-research.md`. Proceed to planning?" — Skip this gate if `Gate-Mode: autonomous`.

### Phase 2: Plan (Architect → Breakdown)

#### Step 2a: Architect (opus)

```
Agent(subagent_type="adlc-planner", description="Architect approach",
  prompt="Project directory: <project-dir>\nMode: architect\nDepth: <depth>\n\nRead 1-research-brief.md. Task: <task>")
```

Produces `2-architecture.md`.

#### Step 2b: Critique (opus, optional)

If `Depth: thorough`, spawn critic in parallel with breakdown:
```
Agent(subagent_type="adlc-planner", description="Critique architecture",
  prompt="Project directory: <project-dir>\nMode: critique\nDepth: <depth>\n\nRead 2-architecture.md. Task: <task>",
  run_in_background=true)
```

Produces `2-architecture-critique.md`. The breakdown agent should read it.

#### Step 2c: Breakdown (sonnet)

```
Agent(subagent_type="adlc-planner", model="sonnet", description="Plan breakdown",
  prompt="Project directory: <project-dir>\nMode: breakdown\nDepth: <depth>\n\nRead 2-architecture.md (and 2-architecture-critique.md if present). Task: <task>")
```

Produces `2-plan.md` + `2-plan-brief.md` + per-step `2-plan-S<N>.md` files.

Present summary (architecture summary, step count, wave count, key risks).

**Gate**: "Plan is ready. Review `<project-dir>/2-plan.md`. Proceed to QA?" — Skip this gate if `Gate-Mode: autonomous`.

### Phase 3: QA (Parallel Criteria + Adversary → Synthesize)

Spawn criteria + adversary in parallel:
```
Agent(subagent_type="adlc-qa", description="QA criteria",
  prompt="Project directory: <project-dir>\nMode: criteria\nDepth: <depth>\n\nRead 2-plan-brief.md. Task: <task>",
  run_in_background=true)

Agent(subagent_type="adlc-qa", model="opus", description="QA adversary",
  prompt="Project directory: <project-dir>\nMode: adversary\nDepth: <depth>\n\nRead 1-research-brief.md and 2-plan-brief.md. Task: <task>",
  run_in_background=true)
```

When both complete:
```
Agent(subagent_type="adlc-qa", description="QA synthesize",
  prompt="Project directory: <project-dir>\nMode: synthesize\nDepth: <depth>\n\nMerge 3-qa-criteria.md and 3-qa-adversary.md.")
```

Produces `3-qa.md` + `3-qa-brief.md` + per-step `3-qa-S<N>.md` files.

Present summary (criteria count, top edge cases, plan gaps).

**Gate**: "QA brief is ready. Review `<project-dir>/3-qa.md`. Proceed to implementation?" — Skip this gate if `Gate-Mode: autonomous`.

### Phase 4: Implement (Parallel Waves + Coder/Tester Split)

Read the Execution Graph from `2-plan-brief.md`.

#### Per-step spawning

For each step in the current wave, choose between:

**Combined mode** (default, simpler): one implementer per step with `Mode: both`.
```
Agent(subagent_type="adlc-implementer", description="Implement S1",
  prompt="Project directory: <project-dir>\nMode: both\nDepth: <depth>\n\nImplement step S1. Task: <task>",
  run_in_background=true)
```

**Split mode** (faster for non-trivial steps): spawn coder + tester for the same step in parallel. Because they run simultaneously, the tester MUST use only `2-plan-S<N>.md` for the interface contract — do NOT instruct it to read in-progress implementation files.
```
Agent(subagent_type="adlc-implementer", description="Code S1",
  prompt="Project directory: <project-dir>\nMode: coder\nDepth: <depth>\n\nImplement step S1 code only. Task: <task>",
  run_in_background=true)

Agent(subagent_type="adlc-implementer", description="Test S1",
  prompt="Project directory: <project-dir>\nMode: tester\nDepth: <depth>\n\nWrite tests for step S1 only. Task: <task>",
  run_in_background=true)
```

Use split mode when the step involves >1 file of implementation AND has >2 acceptance criteria. Otherwise use combined mode.

#### Wave coordination

Execute waves sequentially. Within each wave, launch all step agents (combined or split) in a single message with multiple tool calls so they run concurrently.

**Important**: Parallel steps within a wave MUST NOT modify the same files. Coder + tester for the SAME step can run in parallel because the tester writes test files (different paths) while the coder writes implementation files.

#### Inner fix loop

If a wave finishes with test failures, spawn:
```
Agent(subagent_type="adlc-implementer", description="Fix S<N>",
  prompt="Project directory: <project-dir>\nMode: fixer\nDepth: <depth>\n\nTest failures for step S<N>:\n<failure output>\n\nMake minimal targeted fix. Overwrite the existing `4-implementation-S<N>.md` summary — do not create a new file.")
```

Cap at 2 fix attempts per step. If still failing, escalate by surfacing the failure in the gate message — do NOT proceed to verify.

After all waves complete, read all `4-implementation-*.md` files and present (files changed, tests added, deviations, unmet criteria).

**Gate**: "Implementation complete. Proceed to verification?" — Skip this gate if `Gate-Mode: autonomous`.

### Phase 5: Verify (Parallel Evidence → Synthesize)

Spawn 3-4 evidence-gathering verifiers in parallel:
```
Agent(subagent_type="adlc-verifier", model="sonnet", description="Verify: criteria",
  prompt="Project directory: <project-dir>\nMode: criteria-check\nDepth: <depth>\n\nTask: <task>",
  run_in_background=true)

Agent(subagent_type="adlc-verifier", model="sonnet", description="Verify: tests",
  prompt="Project directory: <project-dir>\nMode: test-run\nDepth: <depth>\n\nTask: <task>",
  run_in_background=true)

Agent(subagent_type="adlc-verifier", model="sonnet", description="Verify: regression",
  prompt="Project directory: <project-dir>\nMode: regression-check\nDepth: <depth>\n\nTask: <task>",
  run_in_background=true)
```

**Conditionally spawn code-review**: Skip only if `Depth: quick`. Otherwise always spawn (git diff --shortstat is unreliable as a threshold since it measures all uncommitted repo changes, not just this ADLC run):
```
Agent(subagent_type="adlc-verifier", model="sonnet", description="Verify: code review",
  prompt="Project directory: <project-dir>\nMode: code-review\nDepth: <depth>\n\nTask: <task>",
  run_in_background=true)
```

When all evidence agents complete, spawn synthesizer (opus):
```
Agent(subagent_type="adlc-verifier", description="Verify: synthesize",
  prompt="Project directory: <project-dir>\nMode: synthesize\nDepth: <depth>\n\nRead all 5-verify-*.md files. Task: <task>")
```

Produces `5-verification.md`.

Present final verdict (status, critical issues, test results, recommendation). Skip the PASS/FAIL gate prompts below if `Gate-Mode: autonomous` — still present the verdict, but do not pause for a response.

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

Run `rm -f <project-dir>/<pattern>` for each matching glob before re-spawning. Tell the user which artifacts are being deleted.

### Re-running phases inside a hierarchical project

The table above still applies unchanged for flat projects and for the Leaf Task Pipeline of an individual stream. For hierarchical projects (where `<project-dir>/project.yaml` exists), it extends as follows rather than being replaced:

- **Scope the glob to the stream, not the whole project**: to re-run a phase for one stream, delete artifacts inside that stream's own directory only — `rm -f <project-dir>/<stream-slug>/<pattern>` — using the same phase-to-pattern mapping from the table above. Sibling streams' directories are never touched by this operation.
- **Reset the stream's status after deleting**: after the glob-delete, the affected stream's `status` field in the **parent** `project.yaml` must be reset to the phase being re-run (e.g. re-running Plan for `stream-a` sets `stream-a`'s `status: plan`, re-running Research sets `status: research`, re-running QA sets `status: qa`, re-running Implement sets `status: implement`, re-running Verify leaves `status` at whatever it already was mid-verify or sets it back to `verify`), using the same atomic `project.yaml.tmp` → `mv` write mechanics as Step 4. This is required so Step 3's skip/blocked logic in the Hierarchical Project Loop doesn't mistake the stream for `completed` and silently skip it on the next wave iteration — without this reset, a re-run would be a no-op from the loop's point of view.
- **Re-running decomposition itself**: to re-run Step 0 (re-decompose a project level from scratch), delete that level's own `project.yaml` (and any stream directories it produced, if starting fully clean) and re-invoke `/adlc <task>` against `<project-dir>` — this routes back through `## Project Mode Detection` as `flat` (no `project.yaml` present) and re-triggers Step 0's decomposition gate. This is a destructive operation for anything already completed under the old decomposition and should be confirmed with the user before deleting.
- **A decomposition-point stream** (one whose own directory has a nested `project.yaml`) cannot be "re-run" via the artifact-glob table at all, since it has no `4-*.md`/`5-*.md` of its own at that level — re-running one of its leaf streams means recursing into its directory and applying this same section there.

## Rules

1. **Always gate on human approval** — never auto-proceed between phases unless the user says "run it all" or "autonomous mode". Exception: the quick path skips gates.
2. **Spawn in parallel** — when launching multiple agents in one phase, put them in ONE message with multiple Agent tool calls so they actually run concurrently.
3. **Wait for completion** — when agents run in background, wait for all to finish before spawning synthesizers or moving to the next wave.
4. **Pass task + project dir + depth + mode** — every agent prompt MUST include all four.
5. **Use briefs downstream** — once `1-research-brief.md` exists, downstream phases read the brief, not the full artifact (unless explicitly noted).
6. **Don't do the work yourself** — invoke phase agents.
