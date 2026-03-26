---
description: Run the full SDLC pipeline (Research → Plan → QA → Implement → Verify)
argument-hint: Task or feature description
---

# SDLC Pipeline

Run the full software development lifecycle for the given task.

Task: $ARGUMENTS

Launch the `sdlc` agent to orchestrate the pipeline:

```
Agent(subagent_type="sdlc", description="SDLC: [task summary]", prompt="$ARGUMENTS")
```
