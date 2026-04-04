---
name: spec-writer
description: Canonical structure and writing rules for SPEC.md — the single source of truth for the lightweight AI-driven development workflow. Covers section definitions, acceptance criteria format, testStrategy classification, and amendment rules.
license: MIT
compatibility: opencode
---

## What I define

The canonical format for `SPEC.md`, the single living document for lightweight workflow.

---

## SPEC.md section structure

Every SPEC.md must contain these sections in this order:

### `## Goal`
One paragraph. The north star. Written in plain language. States what the finished software does for the user, not how it is built. Must be stable — if the goal changes, the whole spec is in question.

### `## Features`
Numbered list. Each item: `N. <feature name> — <acceptance criterion>`.
The acceptance criterion must be **measurable**: a specific HTTP status, a UI state, an error message, a database record. Never "should work" or "displays correctly".

Example:
1. Create todo (title required) → POST /todos returns 201 with { id, title, completed, created_at } 2. List todos with filter → GET /todos?status=active returns 200 [] containing only incomplete todos 3. Toggle complete → PATCH /todos/{id} returns 200 with completed flipped
### `## Out of scope`
Explicit list of things that will NOT be built. This is as important as the feature list — it prevents scope creep and stops the build agent from anticipating requirements.

### `## Architecture`

#### `### Data model`
Table or list of entities with field names, types, constraints, and defaults.

#### `### Interface contracts`
For backend APIs: base URL, each route (method + path → status + response shape). For frontend modules: exported function/component signatures. These contracts are the integration test targets — they must be precise.

#### `### Tech stack`
One line per choice: tool → rationale. Rationale must be one clause ("SQLite → zero-config, no server process for a single-user app").

#### `### Non-goals`
Architecture-level decisions that are explicitly deferred (e.g., "no pagination", "no auth", "no real-time updates"). Different from `## Out of scope` — these are design decisions, not feature exclusions.

### `## Task list`

Organised by phase. Each phase heading: `### Phase N — <phase goal>`.
Each task: `- [ ] <task-id> <task description> [testStrategy]`

Task ID format: `P<phase>-T<task>` (e.g., `P2-T3`).
testStrategy tag: `[tdd]`, `[smoke]`, or `[scaffold]`.

Example:
markdown ### Phase 2 — Backend API - [ ] P2-T1 Todo SQLModel + DB init [tdd] - [ ] P2-T2 POST /todos endpoint [tdd] - [ ] P2-T3 FastAPI app + CORS wiring [smoke]
Tasks within a phase must be ordered so that each task's dependencies are listed above it.
Use `blocked_by: P<n>-T<m>` inline annotation if a task depends on a task from another phase.

### `## Open questions`
Used during the clarification loop only. Each item: `- [ ] <question>`. Mark resolved with `[x]` and append the answer inline. Remove the section entirely once all questions are resolved — do not leave an empty section.

### `## Current phase` (transient)
Written by the build agent at the start of each phase session. Contains the expanded task breakdown and session scratch notes. Removed by the merger agent after phase completion.

### `## DIVERGENCE — <task-id> — <date>` (conditional)
Appended by the reviewer or build agent when implementation cannot satisfy the plan. See the `check-runner` skill for the exact format. Never removed — divergences are a permanent audit record. Resolved divergences are annotated with the human decision and a `[resolved]` marker.

---

## Amendment rules

**Only amend what was flagged.** At the human checkpoint, apply targeted changes to flagged sections only. Do not rewrite sections that were not challenged.

**Propagate changes to all dependents.** If a data model field is removed (e.g., `description`), update: the data model, every interface contract that references it, and every task description that mentions implementing it.

**Dated architecture amendments.** If an interface contract changes after the build loop has started, append a note: `<!-- amended <date>: <what changed and why> -->` inline after the contract entry.

---

## Writing style rules

- **Imperative voice** in task descriptions: "Implement POST /todos", not "Implementation of POST /todos".
- **No filler phrases**: "simple", "basic", "just", "easy" — these carry no information.
- **Concrete nouns**: field names, HTTP methods, status codes, type names — never "the endpoint" or "the data".
- **One fact per line** in interface contracts. Do not combine method + response shape + error codes into one sentence.
