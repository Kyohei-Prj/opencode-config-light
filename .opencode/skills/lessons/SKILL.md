---
name: lessons
description: Record implementation struggles and their solutions to AGENTS.md so future agents can resolve the same problems faster. Use this after completing a task to log any non-trivial obstacles encountered during the implementation.
license: MIT
compatibility: opencode
---

## What I do

After a task is complete, I guide you to append a structured lesson entry to `AGENTS.md` at the repo root. Each lesson captures:

- The task ID and a short title
- What the problem was (the struggle)
- Why it happened (root cause)
- How it was resolved (the fix)
- A pattern name so future agents can search quickly

## When to use me

Call me at the end of every task completion run **if any of the following occurred**:

- A test failed for an unexpected reason (not a missing implementation)
- Had to be repeated more than once due to setup errors
- The implementation required deviating from the SPEC.md interface contract
- More than one implementation attempt was needed for any single test
- A surprising language/framework behaviour was encountered

Do **not** call me for smooth runs where every test failed for the right reason on the first try and passed after one implementation attempt.

## AGENTS.md format

Append the following block to `AGENTS.md`. Create the file if it does not exist.

```markdown
## Lessons

<!-- Each entry is appended by the tdd-lessons skill. Newest entries go at the top. -->

---

### [TASK-ID] <Short title that names the pattern>

**Date:** YYYY-MM-DD  
**Task:** <One-sentence description of the task being implemented>

**Struggle:**  
<What went wrong and at which step (1–6). Be specific — include the error message or symptom.>

**Root cause:**  
<Why it happened. Framework quirk, incorrect assumption, missing fixture, wrong import path, etc.>

**Resolution:**  
<Exactly what was changed to fix it. Code snippets are welcome but keep them short.>

**Pattern:** `<kebab-case-pattern-name>`  
<One-sentence rule a future agent can apply immediately, e.g. "Always mock X before importing Y in this repo.">

---
```

## Step-by-step instructions

1. Open `AGENTS.md` (or create it with an `## Lessons` heading if absent).
2. Find the `## Lessons` section. If it does not exist, append it.
3. Insert the new entry block **immediately after** the `## Lessons` heading and the comment line — newest entries go first.
4. Fill every field. Do not leave placeholders.
5. Keep the `**Pattern:**` tag line a single searchable kebab-case token (e.g. `missing-async-setup`, `interface-contract-deviation`, `fixture-import-order`). Future agents will grep for this.
6. Save the file. Do not stage or commit — that is the caller's responsibility.

## Searching existing lessons

Before starting a new TDD task, an agent may call this skill and then run:

```bash
grep -A 12 "Pattern:" AGENTS.md
```

to get a quick index of known pitfalls. If a pattern tag matches the current task domain, read the full entry before writing any test.
