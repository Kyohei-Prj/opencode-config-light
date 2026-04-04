---
name: tdd-loop
description: Step-by-step TDD inner loop for the lightweight workflow. Covers the full tdd cycle (AAA pattern, failure verification, minimum implementation, refactor under green) and the lighter smoke-test approach for glue code. Reference this before executing any tdd or smoke task.
license: MIT
compatibility: opencode
---

## TDD inner loop (for `tdd` tasks)

The goal of TDD in this workflow is not ceremony — it is to **make the acceptance criterion executable before writing implementation code**, so the test defines the target and the implementation satisfies it, not the reverse.

### Step 1 — Name the test after the acceptance criterion

Derive the test function name from the acceptance criterion in SPEC.md.

Format: `test_<what_it_asserts>` (Python) or `it('<what it asserts>')` (TypeScript).

Example: for P2-T2 "POST /todos returns 201 with created item":
python def test_create_todo_returns_201(client):
This makes failures traceable to the spec without reading the test body.

### Step 2 — Write the test using AAA

**Arrange** — Set up the minimum state needed for the test. Use fixtures for shared setup. Do not put logic in the arrange block.

**Act** — One call to the system under test. One line. No conditionals.

**Assert** — One logical outcome. Multiple `assert` statements are allowed only if they verify the same logical fact (e.g., checking both `status_code == 201` and the response body together is checking one logical outcome: "the endpoint created the resource correctly").
```python
def test_create_todo_returns_201(client, db):
    # Arrange
	payload = {"title": "Buy oat milk"} 
	
	# Act 
	response = client.post("/todos", json=payload) 
	
	# Assert 
	assert response.status_code == 201 data = response.json() 
	assert data["title"] == "Buy oat milk" assert data["completed"] is False 
	assert "id" in data 
	assert "created_at" in data
```
### Step 3 — Confirm the failure is the right failure

Run only the new test. Read the failure message. Confirm:

| Acceptable failure | Not acceptable — fix the test first |
|---|---|
| `404 Not Found` | `SyntaxError` in the test file |
| `ImportError: cannot import name 'router'` | `AssertionError` caused by wrong fixture setup |
| `AttributeError: 'NoneType' object has no attribute 'status_code'` | `TypeError` from wrong test arguments |

The failure must be **because the production code does not exist yet**, not because the test itself is broken.

### Step 4 — Implement the minimum code to pass

Write the smallest amount of production code that makes the test pass. Do not:
- Implement error handling not yet tested
- Anticipate the next task's requirements
- Refactor existing code unless the test requires it

### Step 5 — Confirm the test passes

Run the test. Green means the acceptance criterion is satisfied. If it fails, return to Step 4. After three implementation attempts on the same test, stop and report — do not continue trying different approaches silently.

### Step 6 — Refactor under green

Refactor for clarity and structure **while the test remains green**. Run the test after every change. Rules:
- Extract a helper function only if the same logic appears twice or more.
- Rename for clarity, not preference.
- Do not introduce abstractions that are not yet needed by any existing test.
- Do not add early returns, caches, or optimisations not required to pass the test.

### Step 7 — Repeat per acceptance criterion

Each acceptance criterion in the task maps to at least one test. An acceptance criterion with multiple paths (happy path + error path) maps to multiple tests.

---

## Smoke test approach (for `smoke` tasks)

Smoke tests are not about TDD — they are safety nets against structural breakage. The process is:

1. **Implement the code directly.** No test-first. The code is correct by inspection.

2. **Identify the structural failure mode.** Ask: what is the one way this wiring could silently break? Examples:
   - A missing export: the module compiles but the symbol is undefined at import.
   - A wrong route method: the server starts but the endpoint returns 405.
   - A missing context provider: the component mounts but throws at runtime.

3. **Write one test that catches that specific failure.** The test should pass immediately after implementation. If it does not, the implementation is broken — fix it.

4. **No coverage gate.** Smoke tests do not count toward the `tdd` coverage requirement.

### Smoke test examples

**Route registration (FastAPI)**
```python
def test_smoke_app_starts_and_routes_registered(client): 
    response = client.get("/todos")
	
	assert response.status_code != 404 # route is registered
	```
**Barrel export (TypeScript)**
```typescript
it('smoke: api module exports all expected symbols', async () => {
    const mod = await import('../lib/api');
	
	expect(mod.api).toBeDefined();
	expect(typeof mod.api.list).toBe('function'); 
	}
);
```
**Component render (React)**
```typescript
it('smoke: TodoItem renders without crashing', () => { 
    render(<TodoItem id={1} title="Test" completed={false} onToggle={() => {}} onDelete={() => {}} />);
	
	expect(screen.getByText('Test')).toBeInTheDocument(); 
	}
);
```
---

## What is never acceptable

- A test that passes without running (e.g., `assert True`).
- A test with no assertions.
- A test whose failure message is ambiguous about what broke.
- Implementation code written before the test is confirmed to fail for the right reason.
- A test named `test_it_works` or `test_basic`.
