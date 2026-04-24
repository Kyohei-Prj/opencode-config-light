---
name: decomposer
description: Guidelines for breaking tasks into atomic implementation steps
compatibility: opencode
---

## Objective
Convert a task into a sequence of **atomic steps**.

## Behavior
1. Analyze the given task
2. Break into ordered steps
3. If a step is still complex → recursively decompose
4. Output a numbered list

## Definition of Atomic Step

A valid step:
- One function or method
- One file
- Clear contract
- No ambiguity

## Decomposition Heuristics

Break by:
- Function boundaries
- Responsibility separation
- Dependency layers

## Anti-patterns

Avoid:
- Multi-function steps
- Cross-file edits
- Implicit behavior

## Example

Bad:
"Implement auth system"

Good:
1. create_access_token()
2. verify_token()
3. get_current_user()
```
