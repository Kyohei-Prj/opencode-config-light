---
name: task-classifier
description: Decision rules and borderline examples for classifying a task as tdd, smoke, or scaffold. Used by the spec agent at decomposition time and by the classifier subagent at execution time. The goal is a narrow, auditable classification that cannot silently expand.
license: MIT
compatibility: opencode
---

## Why classification matters

The `testStrategy` field controls which executor runs, whether a coverage gate applies, and whether a test file is written at all. A misclassification in the `scaffold` or `smoke` direction on a logic task leaves that logic permanently untested. Classification must be **explicit, narrow, and decided before implementation pressure applies**.

---

## Decision tree

Apply the questions in order. Stop at the first match.

### Q1 — Does the task produce any importable code or observable runtime behaviour?

**No** → `scaffold`
The task only creates files/directories, installs packages, or initialises static configs. No function is called, no module is imported, no server handles a request as a result of this task alone.

**Yes** → continue to Q2.

### Q2 — Does the task contain any of the following?

- A conditional branch (`if`, `else`, `switch`, ternary, guard clause)
- A data transformation (mapping, filtering, reducing, parsing, serialising)
- I/O (database read/write, filesystem read/write, HTTP call)
- State mutation (variable reassignment, object property update, cache write)
- An algorithm (sorting, searching, hashing, cryptography)
- Input validation (checking types, lengths, formats, required fields)
- Error handling (try/catch, error response shaping, retry logic)
- A typed API contract with non-trivial behaviour (an endpoint that does more than return a static value)

**Yes to any** → `tdd`

**No** → continue to Q3.

### Q3 — Is the task's correctness fully visible by reading the code, but could still fail at runtime due to structural issues (missing export, wrong registration, broken import path, misconfigured middleware)?

**Yes** → `smoke`

Examples of structural failures that smoke tests catch:
- A barrel file that forgets to re-export a symbol
- A route registered under the wrong HTTP method
- A DI binding where the wrong concrete type is mapped
- CORS middleware added to the wrong app instance
- A component that fails to render because a required context provider is missing

---

## Classification table

| Task description | Strategy | Deciding factor |
|---|---|---|
| `mkdir -p backend/app/tests` | scaffold | No code, no behaviour |
| `uv add fastapi sqlmodel` | scaffold | Package installation |
| `Add .env.local with API base URL` | scaffold | Static config file |
| `Write FastAPI app + CORS wiring` | smoke | Structural — correct by inspection, can break at registration |
| `Barrel export from components/index.ts` | smoke | Structural — missing export is a runtime error |
| `Register routes on the FastAPI router` | smoke | Structural — wrong method or path is a runtime error |
| `TodoItem component (render only)` | smoke | Pure render, no conditions or state |
| `POST /todos endpoint` | tdd | I/O + validation + response shaping |
| `GET /todos?status= filter` | tdd | Conditional data transformation |
| `PATCH /todos/{id} toggle` | tdd | State mutation + I/O |
| `API client fetch wrapper` | tdd | Error handling + URL construction (conditional) |
| `AddTodo form validation` | tdd | Input validation (conditional) |
| `Filter bar component` | smoke | Render + prop pass-through, no logic |

---

## Borderline cases

**"It's just a wrapper"** — Wrappers that add error handling, transform the response, or have conditional behaviour are `tdd`. Wrappers that only change the call signature without adding logic are `smoke`.

**"The component has a conditional"** — If the condition controls rendering (show/hide a label, apply a CSS class), it is `smoke`. If the condition gates a side effect or data transformation, it is `tdd`.

**"It's configuration"** — Static config files (`.env`, `pyproject.toml`, `tailwind.config.ts`) with no computed values are `scaffold`. Config that reads environment variables and applies logic based on them is `tdd`.

**"It's just types"** — TypeScript type definitions and Pydantic models with no validators are `scaffold`. Models with validators (`@validator`, `field_validator`, custom `__init__`) are `tdd`.

---

## Logging format in SPEC.md

When the classifier assigns a strategy, the task line in SPEC.md must end with the strategy tag:
- [ ] P2-T2 POST /todos endpoint [tdd]
If assigned at decomposition time by the spec agent, append a parenthetical reason:
- [ ] P2-T3 FastAPI app + CORS wiring [smoke] (structural wiring — no logic)
This reason is the audit trail. At the human checkpoint, review any task where the reason feels thin.

---

## Challenge criteria for human review

Flag any task for human review if:
- It is classified `smoke` but the description mentions validation, filtering, or transformation.
- It is classified `scaffold` but creates a file that will be imported by source code.
- It is classified `tdd` but the description contains only the words "component", "page", or "layout" with no mention of logic.
