---
description: View, edit, or clear ADLC agent memories for this project
argument-hint: "[view|edit|clear] [agent-name]"
---

# ADLC Memory Management

Manage persistent memories stored by ADLC pipeline agents in `.claude/agent-memory/`.

Parse `$ARGUMENTS` to determine the action and optional agent name.

## Valid agent names

adlc-researcher, adlc-planner, adlc-qa, adlc-implementer, adlc-verifier

## Commands

### `view` (default when no arguments or just `/adlc-memory`)

1. Glob `.claude/agent-memory/adlc*/MEMORY.md` to find all ADLC agent memory files
2. For each file found, read it and display with a header: `### <agent-name>` followed by the content and a line count
3. If no memory files exist, say: "No ADLC agent memories found yet. Memories are created automatically when you run the ADLC pipeline."

### `view <agent>` (e.g., `/adlc-memory view adlc-researcher`)

1. Read `.claude/agent-memory/<agent>/MEMORY.md`
2. Display the contents
3. If it doesn't exist, say: "No memories found for **<agent>**. This agent will start building memories on its next ADLC run."

### `edit <agent>` (e.g., `/adlc-memory edit adlc-planner`)

1. Read `.claude/agent-memory/<agent>/MEMORY.md`
2. Display the current contents to the user
3. Ask the user what they want to change (add entries, remove entries, rewrite sections)
4. Apply the edits and write the file back
5. If the file doesn't exist, ask the user if they want to create it with initial content

### `clear` (no agent specified)

1. Ask for confirmation: "This will delete ALL ADLC agent memories (adlc-researcher, adlc-planner, adlc-qa, adlc-implementer, adlc-verifier). Are you sure?"
2. If confirmed, delete all files matching `.claude/agent-memory/adlc*`
3. Report what was deleted

### `clear <agent>` (e.g., `/adlc-memory clear adlc-implementer`)

1. Check if `.claude/agent-memory/<agent>/MEMORY.md` exists
2. If it exists, ask for confirmation: "This will delete **<agent>** memories. Are you sure?"
3. If confirmed, delete the file (or the entire directory)
4. If it doesn't exist, say: "No memories found for **<agent>** — nothing to clear."

## Error handling

- If the agent name is not in the valid list, say: "Unknown agent **<name>**. Valid agents: adlc-researcher, adlc-planner, adlc-qa, adlc-implementer, adlc-verifier"
- If the command is not recognized, show this help text with available commands
