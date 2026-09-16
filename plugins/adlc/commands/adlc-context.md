---
description: View, add, remove, or configure context sources for ADLC agents
argument-hint: "[view|setup|add|remove|edit|clear]"
---

# ADLC Context Source Management

Manage the `context.yaml` file that declares project-specific files (ERDs, PRDs, architecture docs, coding guidelines, etc.) each ADLC agent should read before starting its protocol.

Parse `$ARGUMENTS` to determine the action.

## Workspace Resolution

Resolve the workspace directory first:

```bash
cat ~/.config/adlc/config 2>/dev/null
```

- If the file exists, parse `workspace_dir` from the JSON. Call it `<workspace_dir>`.
- If the file does not exist: "ADLC workspace not configured yet. Run `/adlc` first to set up your workspace, then come back to configure context sources."

The context config file is `<workspace_dir>/context.yaml`.

## Valid Roles

`researcher`, `planner`, `qa`, `implementer`, `verifier`, `all`

`all` means every agent role reads this source.

## Config Schema

```yaml
sources:
  - path: docs/architecture.md
    description: System architecture overview
    roles: [researcher, planner]
  - path: docs/erd/payments.md
    description: Level 1 ERD for payments
    roles: [planner]
```

- `path` — relative to the repo root
- `description` — one-line summary so agents know what they're reading
- `roles` — which agent roles should read this source

## Commands

### `view` (default when no arguments or just `/adlc-context`)

1. Check if `<workspace_dir>/context.yaml` exists
2. If not: "No context sources configured. Run `/adlc-context setup` to get started, or `/adlc-context add` to add individual sources."
3. If it exists, read and parse it. Display sources grouped by role:

```
## Context Sources

### researcher
- `docs/architecture.md` — System architecture overview
- `docs/prd/requirements.md` — Product requirements

### planner
- `docs/architecture.md` — System architecture overview
- `docs/erd/payments.md` — Level 1 ERD for payments
- `docs/prd/requirements.md` — Product requirements

### qa
- `docs/prd/requirements.md` — Product requirements
- `docs/test-standards.md` — Testing conventions

...
```

Expand `all` into every role when displaying.

### `setup`

Interactive first-time setup. Walk the user through configuring sources for each role.

1. Explain: "I'll walk you through configuring context sources for each agent role. For each role, tell me which files (relative to repo root) agents should read. You can skip any role."
2. For each role in order (`researcher`, `planner`, `qa`, `implementer`, `verifier`):
   - Explain what this role does and what kinds of sources help it (see examples below)
   - Use `AskUserQuestion` to ask if the user wants to add sources for this role
   - If yes, ask for file path(s) and a one-line description for each
   - Validate each path exists: `test -f "<path>"` — warn (but don't block) if missing
3. Ask if there are any sources ALL agents should read (e.g., CLAUDE.md, architecture overview)
4. Write the collected sources to `<workspace_dir>/context.yaml` using:
   ```bash
   python3 -c "
   import yaml
   data = {'sources': [...]}
   with open('<workspace_dir>/context.yaml', 'w') as f:
       yaml.safe_dump(data, f, sort_keys=False, default_flow_style=False)
   "
   ```
5. Display the final config and confirm: "Context sources saved to `<workspace_dir>/context.yaml`. These will be injected into agent prompts on every ADLC run."

**Role examples for the setup conversation:**
- **researcher**: "Architecture docs, system overviews, domain glossaries — anything that helps understand the codebase beyond what grep can find"
- **planner**: "ERDs, PRDs, design docs, architectural decision records — documents that define how things should be built"
- **qa**: "Test standards, quality checklists, PRDs with acceptance criteria — documents that define what 'correct' means"
- **implementer**: "Coding guidelines, style guides, API references — documents that define how code should be written"
- **verifier**: "Review checklists, architecture docs, test standards — documents that define what to verify"

### `add`

Add a single source to the config.

1. If `<workspace_dir>/context.yaml` doesn't exist, create it with an empty `sources: []`
2. Ask the user (or parse from arguments if provided inline):
   - File path (relative to repo root)
   - One-line description
   - Which roles should read it (comma-separated, or `all`)
3. Validate the path exists (warn if not)
4. Validate roles are from the valid set
5. Read existing config, append the new source, write back
6. Confirm: "Added `<path>` for roles: <roles>"

### `remove`

1. Read `<workspace_dir>/context.yaml`
2. Display sources with numbered indices:
   ```
   1. `docs/architecture.md` — System architecture overview [researcher, planner]
   2. `docs/erd/payments.md` — Level 1 ERD [planner]
   3. `docs/test-standards.md` — Testing conventions [qa, verifier]
   ```
3. Ask: "Which source(s) to remove? (enter numbers, comma-separated)"
4. Remove selected entries, write back
5. Confirm what was removed

### `edit`

1. Read `<workspace_dir>/context.yaml`
2. Display current contents
3. Ask the user what they want to change (modify descriptions, change roles, update paths)
4. Apply edits and write back
5. Display updated config

### `clear`

1. Ask for confirmation: "This will delete all context source configuration. Agents will revert to ad-hoc discovery. Are you sure?"
2. If confirmed: `rm <workspace_dir>/context.yaml`
3. Report: "Context sources cleared."

## Error Handling

- If PyYAML is not installed: "PyYAML is required. Install it with `pip3 install pyyaml`."
- If the command is not recognized, show help with available commands
- If the config file is malformed, report the parse error and suggest `/adlc-context setup` to recreate it
