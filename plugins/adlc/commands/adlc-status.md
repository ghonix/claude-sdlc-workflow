---
description: Show a read-only dashboard of all ADLC projects, streams, and their current phase/status
argument-hint: "[none]"
allowed-tools: Read, Glob, Bash
---

# ADLC Status Dashboard

Render a read-only tree view of every ADLC project in the configured workspace: top-level projects, their streams (and nested sub-streams, if the project is hierarchical), current phase/status, blocking dependencies, and branch info.

This command is **read-only for v1**. It never modifies `project.yaml`, artifacts, or any project state. If the user asks it to take an action (resume, retry, drill into a specific stream and continue work, etc.), respond that `/adlc-status` is read-only in v1 and point them to `/adlc <task>` to resume or continue that work.

## Step 1: Resolve workspace directory

Run:
```bash
cat ~/.config/adlc/config 2>/dev/null
```

- If the file exists and contains `workspace_dir`, parse and use that value as `<workspace_dir>`.
- If the file does not exist, see "Error handling" below and stop.

## Step 2: Enumerate top-level projects

List every immediate subdirectory of `<workspace_dir>`:
```bash
find <workspace_dir> -mindepth 1 -maxdepth 1 -type d 2>/dev/null
```

Each immediate subdirectory is one top-level project. If the command produces no output, see "Error handling" below and stop. (Prefer `find` over a `*/` glob here — some shells raise a glob-expansion error on an empty directory that a trailing `2>/dev/null` does not suppress.)

## Step 3: Classify and render each project

For each top-level project directory `<workspace_dir>/<slug>/`:

### 3a. Check for `project.yaml`

```bash
test -f "<workspace_dir>/<slug>/project.yaml" && echo hierarchical || echo flat
```

### 3b. Flat/legacy project (no `project.yaml`)

Infer phase from the highest-numbered artifact present, checking in this precedence order (highest first):

1. `5-verification.md` exists → `completed`
2. `4-implementation*.md` exists (any file matching that glob) → `implement`
3. `3-qa*.md` exists → `qa`
4. `2-plan*.md` exists → `plan`
5. `1-research*.md` exists → `research`
6. None of the above exist → `not started`

If the directory exists but contains none of the recognized artifact patterns AND no `project.yaml` (e.g., an empty or unrelated directory), render it as `unknown` / `not started` rather than omitting it or crashing.

Render as:
```
<slug> [flat/legacy]
  phase: <phase> (<artifact-that-triggered-it> exists)
```

Or, if nothing was found:
```
<slug> [unknown]
  phase: not started (no recognized artifacts found)
```

### 3c. Hierarchical project (`project.yaml` present)

Parse it with:
```bash
python3 -c 'import yaml, json; print(json.dumps(yaml.safe_load(open("<workspace_dir>/<slug>/project.yaml"))))'
```

Render the header:
```
<slug> [hierarchical]
  gate: <gate_strategy>
```

Then render each entry in `streams[]` as a branch line, in the order they appear in the file:

```
  |-- <stream-slug> ............ [<status>]<annotation>
```

Formatting rules:
- Branch prefix `|--` at the current indentation depth (one level of two extra spaces per nesting level below the project header).
- Pad the stream slug with `.` characters (space-dot-dot...-space) so the `[status]` brackets roughly align across sibling entries — match the visual density of the example in `2-architecture.md`'s Dashboard section, exact column alignment is not required.
- `[status]` is the stream's `status` field (`pending`, `research`, `plan`, `qa`, `implement`, `verify`, `completed`, `blocked`, etc.) in square brackets.
- If `status == blocked` (or if any entry in `depends_on` is not yet `completed`), append an annotation ` (waiting: <dep-slug>[, <dep-slug>...])` listing the not-yet-completed dependency slugs.
- If additional phase/wave detail is available (e.g., inferred from that stream's own artifacts the same way flat projects are classified in 3b), you may append it in parentheses, e.g. `(wave 2/3)` — this is optional detail, not required to satisfy the schema.
- If the stream itself is a decomposition point — i.e. `test -f "<workspace_dir>/<slug>/<stream-slug>/project.yaml"` succeeds — recurse: after that stream's line, render its own `streams[]` one indentation level deeper, using the exact same rules (branch chars, padding, brackets, waiting annotations). This recursion is **not bounded to one level** — keep recursing for as many nested decomposition points as exist (depth 3+, 4+, etc. must all render correctly). If the stream is a leaf (no nested `project.yaml`), do not recurse further for it; if you want extra detail you may infer its phase using the flat-project rule (3b) applied to `<workspace_dir>/<slug>/<stream-slug>/`. Note (v1 scope): this rendering-time recursion only reflects sub-`project.yaml` files that already exist on disk — the ADLC orchestrator (`/adlc`) does not auto-trigger `Mode: decompose` against a leaf stream, so any depth beyond the root must currently be seeded manually.
- If a stream has a non-empty `branch` field in `project.yaml`, append it at the end of the line as ` [branch: <branch>]` (this is a v2 placeholder field — most v1 projects will have it empty, in which case omit this suffix entirely).

## Step 4: Assemble final output

Print a header, then each top-level project's rendering (flat or hierarchical) in the order returned by `ls`, separated by a blank line:

```
ADLC Projects
==============

<project 1 rendering>

<project 2 rendering>

...
```

## Example output

```
ADLC Projects
==============

add-hierarchical-project-management [hierarchical]
  gate: per-stream
  |-- api-refactor ............ [implement] (wave 2/3)
  |-- schema-design ........... [completed]
  |-- dashboard ............... [blocked] (waiting: api-refactor)
  |   |-- frontend ............ [pending]
  |   |-- backend ............. [pending]
  |-- documentation ........... [pending]

automate-uber-booking-flow [flat/legacy]
  phase: completed (5-verification.md exists)

fare-estimate-client-uses [flat/legacy]
  phase: research (1-research.md exists)
```

## What this command does NOT do (v1 read-only scope)

- No drill-in to view a specific stream's individual artifacts (the user can `cat`/open those files directly).
- No resume-from-here action — it never spawns `adlc-*` agents or touches `project.yaml`.
- No editing, retrying, or unblocking of streams.

If the user asks for any of the above, respond: "`/adlc-status` is read-only in v1 — it only displays status, it doesn't take action. To resume or continue work on a project or stream, run `/adlc <task>`."

## Error handling

- **No `~/.config/adlc/config`**: "No ADLC workspace configured yet. Run `/adlc` once to set up your workspace." Stop — do not prompt to create one.
- **Workspace directory exists but is empty (no subdirectories)**: "No ADLC projects found yet." Stop.
- **A project directory is neither a recognizable flat project (no artifacts matching `1-research*.md` through `5-verification.md`) nor hierarchical (no `project.yaml`)**: render it as `[unknown]` / `not started` per Step 3b — never omit it silently, never crash.
- **`project.yaml` fails to parse** (invalid YAML): render that project as `<slug> [hierarchical, unreadable]` with a one-line note `project.yaml could not be parsed` and move on to the next project — do not stop the whole command.
