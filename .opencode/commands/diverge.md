---
description: Log a divergence to SPEC.md when implementation reality contradicts the plan. Provide a task ID and a brief description of the issue. Creates a DIVERGENCE block in SPEC.md and pauses the build loop until a human decision is recorded.
agent: build
---

Log a divergence for: $ARGUMENTS

Current SPEC.md architecture contracts:
!`awk '/## Architecture/,/## Task list/' SPEC.md`

## Instructions

1. Parse `$ARGUMENTS` as: `<task-id> <description of divergence>`.
   - If only a task ID is given, ask the user for a one-sentence description before proceeding.
   - If no task ID is given, ask which task this divergence relates to.

2. Append the following block to SPEC.md immediately after the task list section:
markdown ## DIVERGENCE — <task-id> — <date> **What was attempted:** <from $ARGUMENTS or user input> **Root cause hypothesis:** <your analysis — be specific, one paragraph> **Impact on architecture contracts:** <which interface contract or design decision is affected, or "none"> **Proposed options:** 1. <option — amend the contract and update affected tasks> 2. <option — change the implementation approach> 3. <option — descope this task> **Awaiting human decision. Do not proceed with this phase until resolved.**
3. Confirm the block was written. Remind the user to annotate the DIVERGENCE block with their chosen option and then run `/phase` to resume.
