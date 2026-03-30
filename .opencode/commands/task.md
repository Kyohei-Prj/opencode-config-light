---
description: Execute a single task by ID (e.g. /task P2-T3). Reads the task's testStrategy from SPEC.md and invokes the correct executor subagent. Use this to re-run a specific task or to step through tasks manually instead of running the full phase loop.
agent: build
subtask: false
---

Execute task: $ARGUMENTS

Current SPEC.md task list:
!`grep -A 3 "\$ARGUMENTS" SPEC.md`

## Instructions

1. Locate task `$ARGUMENTS` in SPEC.md. Extract its description and `testStrategy`.

2. If `testStrategy` is not set, invoke `@classifier` with the task description to determine it. Write the result back to SPEC.md next to the task.

3. Invoke the matching executor:
   - `tdd`      → `@tdd-executor` — pass the task ID and tell it to load SPEC.md for context
   - `smoke`    → `@smoke-executor` — pass the task ID
   - `scaffold` → `@scaffold-executor` — pass the task ID

4. Report the executor's result. Do NOT run check or merge — those are phase-level operations triggered by `/check` and `/phase` respectively.
