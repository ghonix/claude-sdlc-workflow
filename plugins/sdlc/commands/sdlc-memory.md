---
description: View, edit, or clear SDLC agent memories for this project
argument-hint: "[view|edit|clear] [agent-name]"
---

# SDLC Memory Management

Manage persistent memories stored by SDLC pipeline agents in `.claude/agent-memory/`.

Parse `$ARGUMENTS` to determine the action and optional agent name.

## Valid agent names

sdlc, sdlc-researcher, sdlc-planner, sdlc-qa, sdlc-implementer, sdlc-verifier

## Commands

### `view` (default when no arguments or just `/sdlc-memory`)

1. Glob `.claude/agent-memory/sdlc*/MEMORY.md` to find all SDLC agent memory files
2. For each file found, read it and display with a header: `### <agent-name>` followed by the content and a line count
3. If no memory files exist, say: "No SDLC agent memories found yet. Memories are created automatically when you run the SDLC pipeline."

### `view <agent>` (e.g., `/sdlc-memory view sdlc-researcher`)

1. Read `.claude/agent-memory/<agent>/MEMORY.md`
2. Display the contents
3. If it doesn't exist, say: "No memories found for **<agent>**. This agent will start building memories on its next SDLC run."

### `edit <agent>` (e.g., `/sdlc-memory edit sdlc-planner`)

1. Read `.claude/agent-memory/<agent>/MEMORY.md`
2. Display the current contents to the user
3. Ask the user what they want to change (add entries, remove entries, rewrite sections)
4. Apply the edits and write the file back
5. If the file doesn't exist, ask the user if they want to create it with initial content

### `clear` (no agent specified)

1. Ask for confirmation: "This will delete ALL SDLC agent memories (sdlc, sdlc-researcher, sdlc-planner, sdlc-qa, sdlc-implementer, sdlc-verifier). Are you sure?"
2. If confirmed, delete all files matching `.claude/agent-memory/sdlc*`
3. Report what was deleted

### `clear <agent>` (e.g., `/sdlc-memory clear sdlc-implementer`)

1. Check if `.claude/agent-memory/<agent>/MEMORY.md` exists
2. If it exists, ask for confirmation: "This will delete **<agent>** memories. Are you sure?"
3. If confirmed, delete the file (or the entire directory)
4. If it doesn't exist, say: "No memories found for **<agent>** — nothing to clear."

## Error handling

- If the agent name is not in the valid list, say: "Unknown agent **<name>**. Valid agents: sdlc, sdlc-researcher, sdlc-planner, sdlc-qa, sdlc-implementer, sdlc-verifier"
- If the command is not recognized, show this help text with available commands
