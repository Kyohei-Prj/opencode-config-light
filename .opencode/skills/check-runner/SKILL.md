---
name: check-runner
description: How to invoke the project check script, parse its structured JSON output, map findings to targeted fixes, track retry cycles per task, and write DIVERGENCE blocks to SPEC.md after three failed cycles. Used by the reviewer subagent.
license: MIT
compatibility: opencode
---

## What the check script does

The project check script (`make check` or equivalent) runs all quality gates in sequence and writes a structured JSON result. The reviewer never orchestrates individual tools — it calls one command and reads one output.

### Expected output shape
json { "status": "pass" | "fail", "gates": { "lint": { "status": "pass" | "fail", "findings": [] }, "types": { "status": "pass" | "fail", "findings": [] }, "security": { "status": "pass" | "fail", "findings": [] }, "tests": { "status": "pass" | "fail", "passed": 12, "failed": 0, "errors": [], "coverage": { "net_new_lines": 94, "covered": 86, "pct": 91.5, "gate": 80, "status": "pass" | "fail" } } }, "strategy_filter": { "tdd_tasks": ["P2-T1", "P2-T2"], "smoke_tasks": ["P2-T6"], "scaffold_tasks": ["P1-T1"], "coverage_gate_applied_to": "tdd_tasks" } }
If the project does not have a check script yet, the reviewer runs gates individually and synthesises this shape before acting on it.

### Coverage gate scoping

The coverage gate (`pct >= gate`) applies **only to files touched by `tdd` tasks**. Files touched exclusively by `smoke` or `scaffold` tasks are excluded.

To determine which files are in scope, cross-reference the `strategy_filter.tdd_tasks` list with the git diff of the current branch:
bash git diff main --name-only
---

## Acting on findings

### Lint and type findings

Each finding in `gates.lint.findings` or `gates.types.findings`:
json { "file": "app/routers/todos.py", "line": 42, "rule": "E501", "message": "line too long" }
Fix the minimum code to resolve the finding. Re-run `make check` immediately after fixing all findings from one gate before moving to the next.

### Test failures

Each error in `gates.tests.errors`:
json { "test": "test_p2_t2_create_todo_returns_201", "file": "tests/test_todos.py", "error": "AssertionError: 404 != 201" }
For each failing test:
1. Read the test body — understand what it expects.
2. Read the implementation — find the gap.
3. Apply the minimum fix to the production code (not the test, unless the test has a bug that contradicts SPEC.md).
4. Re-run only the failing test to confirm it passes before re-running the full suite.

### Coverage failures

If `gates.tests.coverage.status == "fail"`:
1. Run `coverage report --show-missing` (Python) or `vitest run --coverage` (TypeScript) to identify uncovered lines.
2. Write a test for each uncovered branch. Each test must follow the AAA pattern and be traceable to an acceptance criterion.
3. If a line is unreachable by design (e.g., a defensive `raise` in an impossible path), add a coverage exclusion comment (`# pragma: no cover` / `/* istanbul ignore next */`) with a reason comment above it.

---

## Retry cycle tracking

The retry cycle count tracks **per task**, not per run.

After each `make check` that fails on findings attributable to a specific task, increment that task's retry count. Track it as a comment in SPEC.md next to the task:
- [ ] P2-T2 POST /todos endpoint [tdd] <!-- review-cycles: 2 -->
**After three failed cycles on the same task → write a DIVERGENCE block.**

---

## DIVERGENCE block format

Append to `SPEC.md` after the task list:
markdown ## DIVERGENCE — P2-T2 — 2025-09-14 **What was attempted:** - Cycle 1: Added missing route decorator — tests still failing on response shape - Cycle 2: Fixed response model serialisation — coverage gate failing on error path - Cycle 3: Added error path test — lint failing on import order after refactor **Why check keeps failing:** The endpoint's error response shape does not match SPEC.md's TodoResponse contract. Fixing the shape causes the serialisation test to fail because SQLModel's model_validate does not coerce the created_at field as expected. **Impact on architecture contracts:** TodoResponse in the Interface contracts section may need to specify the created_at format explicitly (ISO 8601 string vs datetime object). **Proposed options:** 1. Amend the TodoResponse contract in SPEC.md to specify created_at: str (ISO 8601) and update P2-T1 (model definition) accordingly. 2. Add a custom serialiser in the model to coerce datetime to string before the Pydantic validation step. 3. Descope created_at from the response contract — return only id, title, completed. **Awaiting human decision. Do not proceed with this phase until resolved.**
---

## Interface contract drift detection

After check passes, compare the implementation's actual API surface to SPEC.md's `### Interface contracts` section:

1. For each contract entry, make a real request (using the test client or `curl`) and compare the response shape to the declared shape.
2. Report any field that is present in the implementation but absent from the contract, or vice versa.
3. **Never amend SPEC.md unilaterally.** Report the drift and ask whether to fix the implementation or update the contract.

---

## When check is not yet set up

If `make check` does not exist, run the following individually and collect output:

**Python (FastAPI) backend:**
bash ruff check app/ tests/ mypy app/ semgrep --config=auto app/ --json pytest --cov=app --cov-report=json tests/
**TypeScript (Next.js) frontend:**
bash npx eslint src/ --format json npx tsc --noEmit npx vitest run --coverage --reporter=json
Synthesise results into the expected JSON shape before acting on them.
