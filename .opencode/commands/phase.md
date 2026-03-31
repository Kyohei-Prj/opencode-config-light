---
description: Start or resume the build loop. Reads SPEC.md, selects the next incomplete phase, creates a feature branch, then executes each task using the appropriate executor subagent (tdd-executor, smoke-executor, or scaffold-executor). Runs reviewer and merger on completion.
agent: build
---

Start the build loop for the next incomplete phase.

Git status:
!`git log --oneline -5`
!`git branch`

## Instructions

1. Read `@SPEC.md` and find the next phase where at least one task is `[ ]` (incomplete). If a `## Current phase` section exists, resume it.

2. Create a feature branch if not already on one:
   `git switch -c phase/<N>-<slug>` where slug is a 2–3 word kebab-case summary of the phase goal.

3. For each incomplete task in the phase, in order:
   a. Invoke `@classifier` with the task description to confirm or assign its `testStrategy`.
   b. Log the strategy next to the task in SPEC.md if not already there.
   c. Invoke the matching executor as a subtask:
      - `tdd`      → `@tdd-executor`
      - `smoke`    → `@smoke-executor`
      - `scaffold` → `@scaffold-executor`
   d. Mark the task done in SPEC.md and move on to next task.

4. When all tasks in the phase are complete, invoke `@reviewer` to run check and apply fixes.

5. If reviewer reports green, invoke `@merger` to squash-merge and update SPEC.md.

6. Report the next unblocked phase or confirm the project is complete.
