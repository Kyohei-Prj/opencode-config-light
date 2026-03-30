# Lightweight AI-Driven Development Workflow for OpenCode

A semi-automated software development workflow built on [OpenCode](https://opencode.ai), an open-source CLI AI coding assistant. The workflow turns vague requirements into a running, tested codebase through two stages: a structured planning stage that produces a single living document (`SPEC.md`), and a build loop that drives implementation task by task using the right level of testing discipline for each task.

---

## Table of contents

- [How it works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [File structure](#file-structure)
- [Quick start](#quick-start)
- [Stage 1 — Spec](#stage-1--spec)
  - [What the spec agent does](#what-the-spec-agent-does)
  - [SPEC.md anatomy](#specmd-anatomy)
  - [The human checkpoint](#the-human-checkpoint)
- [Stage 2 — Build loop](#stage-2--build-loop)
  - [Test strategies](#test-strategies)
  - [How the build loop works](#how-the-build-loop-works)
  - [What the reviewer does](#what-the-reviewer-does)
  - [What the merger does](#what-the-merger-does)
- [Command reference](#command-reference)
- [Agent reference](#agent-reference)
- [Skill reference](#skill-reference)
- [The check script](#the-check-script)
- [Handling divergences](#handling-divergences)
- [Worked example — todo app](#worked-example--todo-app)
- [Adapting the workflow](#adapting-the-workflow)
- [Troubleshooting](#troubleshooting)

---

## How it works

The workflow has two stages and one core principle: **one living document (`SPEC.md`) replaces all planning artifacts**.
You OpenCode │ │ │ /spec <idea> │ ├─────────────────────►│ Stage 1: clarification loop │◄─────────────────────┤ → batched questions │ answers │ ├─────────────────────►│ → writes SPEC.md │◄─────────────────────┤ → flags 2–3 things for review │ approve / annotate │ ├─────────────────────►│ → applies targeted amendments │ │ │ /phase │ Stage 2: build loop (repeats) ├─────────────────────►│ → classifies each task (tdd/smoke/scaffold) │ │ → executes tasks via specialist subagents │ │ → runs check, fixes findings │ │ → squash-merges to main │◄─────────────────────┤ → reports next phase or "done"
Each phase ends with a green `make check` and a squash merge to `main`. `SPEC.md` is updated in place throughout — tasks checked off, divergences logged, contracts amended with dated notes. There is no separate architecture document, no DAG file, no phase state file to keep in sync.

---

## Prerequisites

- [OpenCode](https://opencode.ai/docs/) installed and configured with a model provider
- Git initialised in your project directory (`git init`)
- A `make check` script (or equivalent) that runs lint, type check, and tests — see [The check script](#the-check-script)
- Basic familiarity with the OpenCode TUI (the `/` command prefix and `@` agent mentions)

> **No specific language or framework is required.** The workflow is language-agnostic. The examples in this README use Python/FastAPI and TypeScript/Next.js, but the agents and skills work with any stack.

---

## Installation

### Option A — Project-local (recommended)

Copy the `.opencode/` directory into your project root. The workflow will only be active inside this project.
bash # From the downloaded zip unzip opencode-workflow.zip -d your-project/ cd your-project git add .opencode/ git commit -m "chore: add opencode workflow"
### Option B — Global

Copy the contents of `.opencode/` to your global OpenCode config directory. The workflow will be available in every project.
bash # macOS / Linux cp -r .opencode/agents/* ~/.config/opencode/agents/ cp -r .opencode/commands/* ~/.config/opencode/commands/ cp -r .opencode/skills/* ~/.config/opencode/skills/
> **Prefer project-local.** Global agents and skills apply to every OpenCode session, which can interfere with other projects. Install globally only if you want the workflow available everywhere.

---

## File structure
.opencode/ ├── agents/ │ ├── spec.md # Primary: Stage 1 planning │ ├── classifier.md # Subagent: assigns tdd/smoke/scaffold │ ├── tdd-executor.md # Subagent: strict TDD inner loop │ ├── smoke-executor.md # Subagent: implement + one smoke test │ ├── scaffold-executor.md # Subagent: shell commands + verification │ ├── reviewer.md # Subagent: runs check, fixes, escalates │ └── merger.md # Subagent: git merge, SPEC.md bookkeeping ├── commands/ │ ├── spec.md # /spec — start Stage 1 │ ├── phase.md # /phase — run the build loop │ ├── task.md # /task — execute one task by ID │ ├── check.md # /check — run check only │ └── diverge.md # /diverge — log a divergence manually └── skills/ ├── spec-writer/SKILL.md # SPEC.md section structure + writing rules ├── task-classifier/SKILL.md # tdd/smoke/scaffold decision tree ├── tdd-loop/SKILL.md # AAA pattern + smoke test format ├── check-runner/SKILL.md # check output parsing + retry cycles └── git-workflow/SKILL.md # branch naming, merge, tagging
---

## Quick start

### 1. Initialise git
bash cd your-project git init git commit --allow-empty -m "chore: initial commit"
### 2. Add a check script

Create a `Makefile` at your project root. See [The check script](#the-check-script) for examples per stack.

### 3. Start Stage 1

Open OpenCode in your project directory and run:
/spec Build me a todo web app. Python/FastAPI backend, TypeScript/Next.js frontend, SQLite database.
Or switch to the spec agent first with **Tab**, then describe your requirements directly.

### 4. Answer the clarification questions

The spec agent will ask all its questions in one batch. Answer them, then review the generated `SPEC.md`.

### 5. Approve and start building

Once you are happy with `SPEC.md`, tell the spec agent to proceed. Then run:
/phase
Repeat `/phase` for each subsequent phase. The build loop handles everything inside each phase automatically.

---

## Stage 1 — Spec

### What the spec agent does

The `spec` agent is a **primary agent** (switch to it with **Tab** in the TUI). It has file write access but no bash access — it can only read and write files, never execute commands or touch your codebase.

Its process:

1. **Read first, ask second.** It reads your full input and builds a model of what is known versus unknown before asking anything. It never fires questions immediately.
2. **One clarification round.** All questions arrive in a single numbered list. It never asks follow-up questions after your answers — if an answer creates a new ambiguity, it folds that into a targeted amendment before writing.
3. **Write `SPEC.md` in one pass.** After the clarification round resolves, it loads the `spec-writer` and `task-classifier` skills and writes the complete document.
4. **Flag, not annotate.** It surfaces two or three specific things worth human review — not a full walkthrough of every decision.

### SPEC.md anatomy

`SPEC.md` has a fixed section order. Every section serves a specific purpose:
markdown # SPEC.md — <project name> ## Goal One paragraph. The north star. What the finished software does for the user. ## Features Numbered list. Each item has a measurable acceptance criterion. 1. Create todo (title required) → POST /todos returns 201 with { id, title, completed, created_at } 2. List todos with filter → GET /todos?status=active returns 200 [] of incomplete todos 3. Toggle complete → PATCH /todos/{id} returns 200 with completed flipped 4. Delete todo → DELETE /todos/{id} returns 204 5. Filter selection persists in URL query param ## Out of scope Explicit exclusions. As important as the feature list. - Authentication and multi-user support - Due dates, priority, or labels - Real-time updates - Pagination ## Architecture ### Data model todos: id (int PK), title (str, required), completed (bool, default false), created_at (datetime, auto) ### Interface contracts Base: http://localhost:8000 POST /todos → 201 TodoResponse GET /todos?status= → 200 TodoResponse[] (status: all | active | completed) PATCH /todos/{id} → 200 TodoResponse DELETE /todos/{id} → 204 TodoResponse { id: int, title: str, completed: bool, created_at: str (ISO 8601) } ### Tech stack FastAPI → lightweight, async-native, auto-generates OpenAPI docs SQLModel → single model definition for DB schema and Pydantic validation SQLite → zero-config, no server process, appropriate for single-user app Next.js 14 (App Router) → file-based routing, server components where useful TailwindCSS → utility-first, no design system dependency ### Non-goals No pagination, no auth, no real-time updates, no mobile app ## Task list ### Phase 1 — Scaffold - [ ] P1-T1 Initialise backend directory structure and pyproject.toml [scaffold] - [ ] P1-T2 Install backend dependencies (fastapi, uvicorn, sqlmodel, pytest, httpx) [scaffold] - [ ] P1-T3 Initialise Next.js app with TypeScript and Tailwind [scaffold] - [ ] P1-T4 Add .env.local with NEXT_PUBLIC_API_URL [scaffold] ### Phase 2 — Backend API - [ ] P2-T1 Todo SQLModel definition and database initialisation [tdd] - [ ] P2-T2 POST /todos endpoint [tdd] - [ ] P2-T3 GET /todos with status filter [tdd] - [ ] P2-T4 PATCH /todos/{id} toggle complete [tdd] - [ ] P2-T5 DELETE /todos/{id} [tdd] - [ ] P2-T6 FastAPI app instantiation and CORS middleware wiring [smoke] ### Phase 3 — Frontend - [ ] P3-T1 Typed API client module [tdd] - [ ] P3-T2 TodoItem component [smoke] - [ ] P3-T3 AddTodo form with title validation [tdd] - [ ] P3-T4 Filter bar component [smoke] - [ ] P3-T5 TodoList component [smoke] - [ ] P3-T6 Wire up main page [smoke]
**Key rules about `SPEC.md`:**

- It is **never fully rewritten** — only flagged sections are amended.
- Architecture contract changes are **dated inline**: `<!-- amended 2025-09-14: removed description field -->`
- Completed tasks are marked `[x]` by the merger after each phase merge.
- Divergences are appended as permanent records — never deleted.

### The human checkpoint

After `SPEC.md` is written, the spec agent flags two or three things and waits. This is your only structured review point before building starts. Three paths forward:

| Your response | What happens |
|---|---|
| **Approve** | Build loop starts. The spec agent confirms readiness. |
| **Annotate specific sections** | The spec agent applies targeted amendments to flagged sections only, then re-confirms. |
| **Reject and redirect** | Describe the fundamental issue. The spec agent rewrites from `requirements.md` with your feedback as constraints. |

**What to look for at the checkpoint:**

- Does the Goal section match your actual intent?
- Are acceptance criteria concrete and measurable — not "works correctly" but a specific status code, value, or UI state?
- Do any `[smoke]` or `[scaffold]` tasks look like they contain logic? (See [Test strategies](#test-strategies).)
- Are the Out of scope and Non-goals sections explicit enough to prevent drift?
- Are the interface contracts precise enough to serve as integration test targets?

---

## Stage 2 — Build loop

### Test strategies

Every task in `SPEC.md` carries a `testStrategy` tag. This tag controls which executor subagent runs and whether a coverage gate applies.

| Strategy | What it covers | Test approach | Coverage gate |
|---|---|---|---|
| `[scaffold]` | Dirs, packages, static configs, CI templates | Shell verification only (`ls`, `which`, `cat`) | None |
| `[smoke]` | Glue code, route registration, DI wiring, barrel exports, pure render components | One integration smoke test (app starts / endpoint responds / component renders) | None |
| `[tdd]` | Any logic: conditionals, I/O, state mutation, validation, transformation, algorithms | Failing test first (AAA) → implement → refactor under green | ≥ 80% on net-new lines |

**The decision tree** (applied by the `classifier` subagent and the `spec` agent):
Does the task produce importable code or observable runtime behaviour? No → scaffold Yes → Does it contain a conditional, I/O, transformation, validation, state mutation, algorithm, or error handling? Yes → tdd No → Is it correct by inspection but could break structurally (missing export, wrong route method, misconfigured middleware)? Yes → smoke
**Borderline cases worth knowing:**

- A React component with a conditional that controls rendering (show/hide a label, apply a CSS class) is `smoke`. A conditional that gates a side effect or data fetch is `tdd`.
- A Pydantic/SQLModel model with no validators is `scaffold`. One with `field_validator` or `model_validator` is `tdd`.
- An API client "wrapper" that adds error handling or URL construction logic is `tdd`. One that only renames the call signature is `smoke`.
- Classification is set at planning time, not at implementation time. If OpenCode disagrees with an assigned strategy during execution, it reports this — it does not silently reclassify and proceed.

### How the build loop works

Running `/phase` starts an automated loop:
For each incomplete task in the current phase (in order): 1. @classifier confirms the testStrategy 2. The matching executor subagent runs: tdd → @tdd-executor (test-first, AAA, refactor) smoke → @smoke-executor (implement, one smoke test) scaffold → @scaffold-executor (shell commands, shell verify) 3. Executor reports completion When all tasks complete: 4. @reviewer runs make check - Parses JSON result - Applies targeted fixes - Re-runs check (up to 3 cycles per task) - Writes DIVERGENCE block if escalation threshold reached 5. @merger squash-merges to main, tags, updates SPEC.md, runs regression gate 6. Reports next phase or "all phases complete"
The loop runs entirely within the OpenCode session. You can watch progress in the TUI's subagent panel. Child sessions created by subagents are accessible with `<Leader>+Down`.

**Resuming an interrupted session:** if a session ends mid-phase (network drop, manual interrupt, context limit), run `/phase` again. It reads `SPEC.md`'s task list and the current git branch to determine where it left off and resumes from the first incomplete task.

### What the reviewer does

After all tasks in a phase complete, the `reviewer` subagent runs automatically. It:

1. Runs `make check` and parses the JSON result.
2. Maps each failing gate's findings to targeted fixes — file, line, rule, and minimum change.
3. Applies the fixes and re-runs check.
4. Tracks retry cycles per task. After **three failed cycles on the same task**, it stops applying fixes and writes a `DIVERGENCE` block to `SPEC.md` instead.
5. After check passes, compares the implemented API surface against `SPEC.md`'s interface contracts. Any drift is reported explicitly — the reviewer never silently amends contracts.

**Coverage gate scoping:** the ≥ 80% coverage gate applies only to files touched by `[tdd]` tasks. Files touched by `[smoke]` or `[scaffold]` tasks are excluded, determined by cross-referencing the `strategy_filter` in the check JSON against the phase branch's git diff.

### What the merger does

After the reviewer confirms a green check, the `merger` subagent:

1. Runs `make check` one final time on the feature branch (belt-and-suspenders).
2. Squash-merges to `main` with a conventional commit message derived from SPEC.md's phase heading.
3. Tags the commit as `phase-N`.
4. Runs the full test suite on `main` (the regression gate).
5. If the regression gate is green: marks all phase tasks `[x]` in `SPEC.md`, removes the `## Current phase` section, and reports the next unblocked phase.
6. If the regression gate is red: stops immediately, does not modify `SPEC.md`, writes a DIVERGENCE entry, and waits for human decision.

---

## Command reference

All commands are typed in the OpenCode TUI with a `/` prefix.

### `/spec <requirements>`

Starts Stage 1. Switches to the `spec` primary agent and begins the clarification loop.
/spec Build me a REST API for a bookmarks manager. Python backend, PostgreSQL, JWT auth.
You can also paste multi-line requirements as `$ARGUMENTS` — just type `/spec` then paste your text. The spec agent reads everything before asking any questions.

**When to use:** at the very start of a project, before any code exists.

---

### `/phase`

Starts or resumes the build loop for the next incomplete phase. Reads `SPEC.md` and git state to determine where to begin.
/phase
No arguments. Run this once per phase. When a phase completes, run it again for the next one.

**When to use:** after the human checkpoint approves `SPEC.md`, and after each phase completes.

---

### `/task <task-id>`

Executes a single task by ID without running the full phase loop. Useful for re-running a specific failed task or stepping through tasks manually.
/task P2-T3
The command reads the task's `testStrategy` from `SPEC.md`, invokes `@classifier` to confirm it if missing, then dispatches to the correct executor. It does **not** run check or merge — those are phase-level operations.

**When to use:** after a divergence is resolved and you want to re-run only the affected task, or when you want finer control than the full `/phase` loop.

---

### `/check`

Runs `make check` via the `reviewer` subagent. Interprets the JSON result, applies targeted fixes, and reports green or blocked.
/check
Safe to run at any time. Does not merge, does not modify `SPEC.md` (unless a divergence threshold is reached).

**When to use:** after manually editing code outside the build loop, or to spot-check gate status mid-phase.

---

### `/diverge <task-id> <description>`

Manually logs a divergence to `SPEC.md` when implementation reality contradicts the plan. Use this when you notice the conflict yourself before the reviewer escalates automatically.
/diverge P2-T2 SQLModel's created_at field serialises as a datetime object, not an ISO string — conflicts with the TodoResponse contract
The command appends a `## DIVERGENCE` block to `SPEC.md` with the task ID, date, a root cause analysis, and proposed options. The build loop will not proceed past this phase until the block is annotated with a human decision.

**When to use:** whenever you realise an interface contract, data model, or design decision in `SPEC.md` needs to change based on what you've learned during implementation.

---

## Agent reference

Agents are invoked automatically by the commands and by each other. You can also invoke subagents directly with `@` in any message.

### `spec` — primary agent

The Stage 1 planning agent. Runs the clarification loop and produces `SPEC.md`. Has write access to files but no bash access — it cannot modify your codebase. Switch to it with **Tab** or start a message with `/spec`.

**Tools:** file read, file write. No bash, no web fetch.

---

### `classifier` — subagent (hidden)

Assigns `tdd`, `smoke`, or `scaffold` to a task description by applying the canonical decision tree from the `task-classifier` skill. Returns a strategy and a one-sentence justification. Called by the build agent before each task executes.

Hidden from the `@` autocomplete — invoked programmatically. You can still call it directly: `@classifier Implement the PATCH /todos/{id} endpoint`.

**Tools:** read-only. No file edits, no bash.

---

### `tdd-executor` — subagent (hidden)

Implements one `[tdd]` task using strict test-driven development. Writes the failing test first (AAA pattern), confirms the failure is for the right reason, implements the minimum code to pass, then refactors under green.

**Tools:** file read, file write, bash (no git push or merge).

---

### `smoke-executor` — subagent (hidden)

Implements one `[smoke]` task directly, then writes a single integration-level smoke test. No test-first, no AAA, no coverage gate. If it encounters conditional logic or data transformation during implementation, it stops and flags a potential misclassification.

**Tools:** file read, file write, bash (no git push or merge).

---

### `scaffold-executor` — subagent (hidden)

Executes one `[scaffold]` task using shell commands. Creates directories, installs packages, writes static config files. Verifies success with `ls`, `which`, or `cat`. Writes no test files.

**Tools:** file read, file write, bash (restricted — no destructive commands).

---

### `reviewer` — subagent (hidden)

Runs `make check`, parses the JSON result, applies targeted fixes, tracks retry cycles per task, and escalates to a `DIVERGENCE` block after three failed cycles. Also performs interface contract drift detection after check passes.

**Tools:** file read, file write, bash (no git push or merge).

---

### `merger` — subagent (hidden)

Squash-merges the phase branch to `main`, applies a conventional commit and tag, runs the regression gate, marks tasks complete in `SPEC.md`, and identifies the next unblocked phase.

**Tools:** file read, file write, bash (no force push or hard reset without confirmation).

---

## Skill reference

Skills are reusable instruction sets loaded on demand by agents. You do not call them directly — agents load them automatically when needed. They are listed here so you know what knowledge each agent draws on.

### `spec-writer`

Defines the canonical `SPEC.md` section structure, acceptance criteria format, task ID and testStrategy tag format, amendment rules, and writing style. Loaded by the `spec` agent before writing `SPEC.md`.

### `task-classifier`

Contains the full decision tree, a classification table of ~15 examples, borderline case analysis, the correct SPEC.md logging format, and challenge criteria for human review. Loaded by the `spec` agent at decomposition time and by the `classifier` subagent at execution time.

### `tdd-loop`

Step-by-step TDD inner loop: naming tests after acceptance criteria, writing AAA tests, verifying the failure is for the right reason, implementing minimum code, refactoring under green. Also covers the smoke test approach for `[smoke]` tasks. Loaded by `tdd-executor` and `smoke-executor`.

### `check-runner`

How to invoke `make check`, the expected JSON output shape, coverage gate scoping, how to act on lint/type/test findings, retry cycle tracking, the DIVERGENCE block format, and interface contract drift detection. Loaded by `reviewer`.

### `git-workflow`

Branch naming conventions, the full pre-merge checklist, squash merge sequence, conventional commit format, phase tag format, regression gate rules, and SPEC.md bookkeeping after merge. Loaded by `merger`.

---

## The check script

The workflow requires a `make check` command at your project root that runs all quality gates and exits with a structured JSON result. If `make check` is not found, the reviewer falls back to running tools individually and synthesising the result — but having a proper script is strongly recommended for reliability.

### Python / FastAPI (backend)
makefile # Makefile .PHONY: check check: @ruff check app/ tests/ --output-format json > /tmp/lint.json 2>&1; LINT_EXIT=$$?; \ mypy app/ --output json > /tmp/types.json 2>&1; TYPE_EXIT=$$?; \ pytest --cov=app --cov-report=json --tb=json tests/ > /tmp/tests.json 2>&1; TEST_EXIT=$$?; \ python scripts/merge_check.py /tmp/lint.json /tmp/types.json /tmp/tests.json > result.json; \ cat result.json; \ [ $$LINT_EXIT -eq 0 ] && [ $$TYPE_EXIT -eq 0 ] && [ $$TEST_EXIT -eq 0 ]
### TypeScript / Next.js (frontend)
makefile check: @npx eslint src/ --format json > /tmp/lint.json 2>&1; LINT_EXIT=$$?; \ npx tsc --noEmit --pretty false > /tmp/types.json 2>&1; TYPE_EXIT=$$?; \ npx vitest run --coverage --reporter=json > /tmp/tests.json 2>&1; TEST_EXIT=$$?; \ node scripts/merge_check.js /tmp/lint.json /tmp/types.json /tmp/tests.json > result.json; \ cat result.json; \ [ $$LINT_EXIT -eq 0 ] && [ $$TYPE_EXIT -eq 0 ] && [ $$TEST_EXIT -eq 0 ]
### Minimal working version (run tools, print pass/fail)

If you want to get started before building a full check script, this minimal version works with the reviewer's fallback logic:
makefile check: ruff check app/ tests/ mypy app/ pytest --cov=app --cov-report=term-missing tests/
The reviewer will run these sequentially and interpret their exit codes and stdout. It won't have structured JSON but will still catch failures.

---

## Handling divergences

A divergence is when implementation reality contradicts something in `SPEC.md` — a data model assumption, an interface contract, a tech stack choice, or a design decision. Divergences are normal. The workflow treats them as explicit planning events, not implementation surprises to silently paper over.

### Automatic escalation

The `reviewer` writes a `DIVERGENCE` block automatically after three failed check cycles on the same task. The block contains:
- What was attempted in each cycle
- Why check keeps failing
- Which interface contract or design decision is affected
- Proposed options with concrete next steps

### Manual logging

You can log a divergence yourself at any time with `/diverge`:
/diverge P3-T1 fetch() in Next.js App Router requires a different error handling pattern than the typed wrapper we designed — the SPEC.md API client contract assumes a plain fetch, but server components use a cached version
### Resolving a divergence

1. Read the `## DIVERGENCE` block in `SPEC.md`.
2. Choose one of the proposed options (or write your own).
3. Annotate the block with your decision:
markdown **Human decision (2025-09-14):** Proceed with option 2 — add a custom serialiser in the model. The TodoResponse contract remains unchanged. **[resolved]**
4. Run `/task <task-id>` to re-execute the task with the new approach, or `/phase` to resume the full loop.

The divergence block is **never deleted** from `SPEC.md`. Resolved divergences serve as a permanent audit record of why the plan changed.

---

## Worked example — todo app

This is a condensed walkthrough of the simulation covered in the workflow design session.

### Stack
- Backend: Python 3.12, FastAPI, SQLModel, SQLite, pytest + httpx
- Frontend: TypeScript, Next.js 14 (App Router), TailwindCSS, Vitest

### Session 1 — Spec
/spec Build me a todo web app. Python/FastAPI backend, TypeScript/Next.js + Tailwind frontend. SQLite database. Keep it simple.
The spec agent identifies five unknowns and asks them in one round:

1. Single-user or multi-user?
2. Operations: create and complete only, or also edit?
3. Fields beyond a title?
4. Should completed todos stay visible?
5. Pagination?

After answering, `SPEC.md` is written with 3 phases and 16 tasks. The agent flags two things:
- `P3-T1` (API client) is `[tdd]` — covers URL construction and error handling. Does this seem right?
- `P3-T3` (AddTodo form) splits: validation logic is `[tdd]`, rendering is `[smoke]`. OK?

You approve both and ask to remove the description field. The spec agent applies the change across all three affected sections (features, data model, interface contract) and confirms.

### Session 2 — Phase 1 (scaffold)
/phase
All four tasks are `[scaffold]`. The `scaffold-executor` creates the directory structure, installs dependencies, runs `create-next-app`, and adds `.env.local`. Each step is verified with a shell check. No test files are written. The phase squash-merges with `feat(phase-1): scaffold project structure and dependencies`.

### Session 3 — Phase 2 (backend)
/phase
Five `[tdd]` tasks, one `[smoke]` task. Example TDD cycle for `P2-T2`:

1. `tdd-executor` writes the failing test for `POST /todos`. Confirms it fails with `404 Not Found` (correct failure reason — the route does not exist yet).
2. Implements the endpoint using SQLModel. Test passes.
3. Adds a second test for missing title → 422. Implements Pydantic validation. Test passes.
4. Refactors: extracts the session dependency. Both tests remain green.

After all six tasks complete, `reviewer` runs `make check`. Result: 12 tests passing, 91% coverage on `[tdd]` files, no lint or type errors. Merger squash-merges with `feat(phase-2): implement backend API with SQLite persistence` and tags `phase-2`.

### Session 4 — Phase 3 (frontend)
/phase
Three `[tdd]` tasks (API client, form validation, filter URL logic) and three `[smoke]` tasks (TodoItem, TodoList, main page wiring). The `smoke-executor` implements each component directly and writes one render smoke test. Check passes. Project complete.

Final state on `main`:
- 3 phase commits, 3 tags
- 20 tests: 12 backend, 8 frontend
- 88% coverage on `[tdd]` files
- 0 lint errors, 0 type errors

---

## Adapting the workflow

### Different language / framework

The agents are language-agnostic. The only thing that changes is the `make check` script. The spec agent will derive tech stack from your `SPEC.md` architecture section and the executors will use the appropriate tools (pytest vs vitest, ruff vs eslint, mypy vs tsc).

### No TDD preference

If you want to disable TDD entirely and use smoke tests for everything, change the `task-classifier` agent's classification rules. You can also override per-task by manually editing the `[strategy]` tag in `SPEC.md` before running `/phase`.

### Larger projects with real parallelism

Add `blocked_by: P<n>-T<m>` annotations to tasks in `SPEC.md`:
- [ ] P3-T1 Typed API client module [tdd] blocked_by: P2-T6
The build agent reads these and will not start a blocked task until the dependency is marked `[x]`. For genuine parallel work across multiple engineers, consider graduating to the structured workflow (with a full DAG) rather than adapting this one.

### Tighter human oversight

If you want to review each task before it executes rather than running the full phase loop, use `/task` instead of `/phase`:
/task P2-T1 /task P2-T2 /check
This gives you line-by-line control while still using the executor subagents and check tooling.

### Relaxing the coverage gate

Edit the `check-runner` skill's coverage section. Change the `gate: 80` threshold, or add an exclusion list of paths that should not count toward coverage.

---

## Troubleshooting

### The spec agent writes SPEC.md immediately without asking questions

Your requirements were specific enough that the agent found no ambiguities. This is fine — review the generated SPEC.md at the checkpoint and annotate anything that needs clarification.

### `/phase` runs but produces no output

Check that `SPEC.md` exists and has at least one `[ ]` task. If all tasks are already `[x]`, the project is complete. If `SPEC.md` is missing, run `/spec` first.

### A skill is not being loaded

Verify the directory name matches the `name` field in the skill's frontmatter exactly. Check that `SKILL.md` is in all caps. Confirm there are no skill name conflicts across global and project-local paths. Run `opencode` with verbose logging to see skill discovery output.

### `make check` is not found

The reviewer falls back to running tools individually. To prevent this, create a `Makefile` in your project root using the examples in [The check script](#the-check-script).

### An executor keeps producing the wrong test strategy

Run `@classifier <task description>` manually to see the decision and its justification. If the justification is wrong, the task description in `SPEC.md` may be ambiguous. Reword it to be more specific about whether the task contains logic.

### The build loop stalls at a divergence

Read the `## DIVERGENCE` block in `SPEC.md`. Annotate it with your chosen option (see [Resolving a divergence](#resolving-a-divergence)). Then run `/task <task-id>` to retry the specific task.

### Regression gate fails after merge

This is the highest-priority issue. Do not run `/phase`. Read the failing test name, locate which phase introduced the regression using `git diff phase-<N-1> phase-<N>`, and run `/diverge regression <description>`. Decide whether to revert with `git revert` or fix forward on a new phase branch.

---
