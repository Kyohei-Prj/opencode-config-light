---
description: Run the project check script and analyse the result. Invokes the reviewer subagent to interpret findings, apply targeted fixes, and detect interface contract drift. Safe to run at any time — does not merge or modify SPEC.md unless a divergence is detected.
agent: build
subtask: true
---

Run the project check script and analyse the output.

Current branch:
!`git branch --show-current`

SPEC.md task strategies (for coverage gate scoping):
!`grep -E "\[(tdd|smoke|scaffold)\]" SPEC.md`

## Instructions

Invoke `@reviewer` with the above context. The reviewer will:
1. Run `make check`
2. Parse the JSON result
3. Apply targeted fixes for any failing gates
4. Detect interface contract drift against SPEC.md
5. Escalate to a DIVERGENCE block in SPEC.md after three failed cycles

Report the reviewer's final status: green (ready to merge) or blocked (divergence logged).
