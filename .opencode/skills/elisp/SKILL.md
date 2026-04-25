---
name: elisp-reliable
description: Guardrails for writing, loading, and testing Emacs Lisp (elisp) code with high reliability. Covers parenthesis auditing, load-path resolution, ERT self-correction loops, and safe interactive evaluation patterns. Use whenever writing, debugging, or testing any .el file or elisp snippet.
license: MIT
metadata:
  version: "1.0"
  audience: AI agents writing Emacs Lisp
  tags: emacs, elisp, ert, testing, lisp
---

# Elisp Reliability Skill

This skill defines strict protocols that an AI agent **must** follow when writing,
loading, and testing Emacs Lisp code. Deviation from these protocols is the primary
cause of load errors, broken ERT suites, and symbol-void failures.

---

## 1. Syntactic Integrity — The Parenthesis Audit

### Small Code Block Scope

Whenever writing or editing elisp, output small code block at a time.
Write single code block or function for single writing action to mitigate unblocked parenthesis error.
Never write multiple functions in one writing action.

### Rule: One Top-Level Form Per Block

Never write deeply nested multi-form blocks in a single code fence.
Split each `defun`, `defvar`, `defcustom`, and `ert-deftest` into its own
top-level form with a blank line between them.

**Bad — avoid this:**
```elisp
(defun foo (x) (if (> x 0) (progn (message "pos") (+ x (let ((y (* x 2))) (- y 1)))) (message "neg")))
```

**Good — do this instead:**
```elisp
(defun foo--compute (x)
  (let ((y (* x 2)))
    (- y 1)))

(defun foo (x)
  (if (> x 0)
      (progn
        (message "pos")
        (+ x (foo--compute x)))
    (message "neg")))
```

### The Parenthesis Audit Protocol

Before outputting any elisp code block, the agent **must** mentally execute
this audit:

1. **Count open vs. close parens.** Every `(` must have a matching `)`.
   Use a running counter: increment on `(`, decrement on `)`.
   The counter must reach exactly `0` at the end of the form.

2. **Check string and comment boundaries.** A `;` starts a comment only
   outside a string. A `"` that is not preceded by `\` opens/closes a string.
   Parens inside strings and comments do not count.

3. **Verify `let` binding structure.** Every `let` or `let*` takes exactly
   two parts: `(BINDINGS BODY...)`. The BINDINGS clause is itself a list of
   `(VAR VAL)` pairs. A missing inner paren here is the #1 source of
   `Wrong type argument: listp` errors.

   ```elisp
   ;; WRONG — missing inner paren around binding pair
   (let (x 0)
     x)

   ;; CORRECT
   (let ((x 0))
     x)
   ```

4. **Verify `cond` clause structure.** Each `cond` clause is `(TEST BODY...)`.
   A bare test without wrapping parens is a syntax error.

5. **Sign off.** Only output the code after the audit passes.

---

## 2. Environment & Path Reliability — Load Guardrails

### 2.1 Always Use `expand-file-name`

Never hardcode absolute paths. Use `expand-file-name` with a base anchor:

```elisp
;; Resolve relative to the current file's directory
(defvar my-pkg-dir
  (file-name-directory (or load-file-name buffer-file-name)))

(add-to-list 'load-path (expand-file-name "lib" my-pkg-dir))
```

For scripts run via `emacs --batch --load`:

```elisp
;; At the top of the entry-point file
(add-to-list 'load-path
             (expand-file-name
              (file-name-directory load-file-name)))
```

### 2.2 Guard Every `require` with `featurep`

Before calling `require`, check whether the feature is already loaded
to avoid redundant file I/O and double-eval side effects:

```elisp
(unless (featurep 'my-feature)
  (require 'my-feature))
```

For optional third-party dependencies:

```elisp
(defun my-pkg--require-optional (feature)
  "Load FEATURE softly; return t on success, nil on failure."
  (condition-case err
      (progn (require feature) t)
    (file-error
     (message "Optional feature `%s' not found: %s" feature err)
     nil)))
```

### 2.3 `load-path` Modification Checklist

Run through this list before any `require` that targets a non-standard path:

| Step | Check |
|------|-------|
| 1 | `load-path` modified **before** the `require` call, not after |
| 2 | Path is an **absolute** string (use `expand-file-name`) |
| 3 | Target `.el` file provides the expected `(provide 'feature-name)` at its end |
| 4 | Feature symbol matches filename: `my-feature` → `my-feature.el` |
| 5 | No circular `require` chains between files in the package |

### 2.4 Batch-Mode vs. Interactive Loading

When generating code that runs in both contexts, guard with `noninteractive`:

```elisp
(unless noninteractive
  ;; Only in interactive Emacs sessions
  (add-hook 'after-init-hook #'my-pkg-setup))
```

---

## 3. Test-Driven Development — The ERT Self-Correction Loop

### 3.1 Test File Structure Template

Every ERT test file must follow this skeleton exactly:

```elisp
;;; my-feature-tests.el --- ERT tests for my-feature  -*- lexical-binding: t; -*-

;; Ensure the feature under test is loaded
(require 'ert)

(add-to-list 'load-path
             (expand-file-name
              (file-name-directory (or load-file-name buffer-file-name))))
(require 'my-feature)

;;; Tests

(ert-deftest my-feature-test-basic ()
  "Describe what this test asserts."
  (should (equal (my-feature-fn 1) 2)))

(ert-deftest my-feature-test-edge ()
  "Edge case: nil input."
  (should-not (my-feature-fn nil)))

;;; my-feature-tests.el ends here
```

### 3.2 Running Tests

**Interactive (inside Emacs):**
```elisp
;; Run all tests matching a prefix
(ert "^my-feature-")

;; Run a single named test
(ert 'my-feature-test-basic)
```

**Batch (from shell — CI-friendly):**
```bash
emacs --batch \
      --load my-feature-tests.el \
      --funcall ert-run-tests-batch-and-exit
```

### 3.3 The Self-Correction Loop

When a test fails, the agent **must** categorize the error before attempting a fix.
Use this decision tree:

---

#### Error Class A: `Symbol's value as variable is void: FOO`

**Cause:** A variable is referenced before it is defined, or `defvar`/`defcustom`
was never evaluated.

**Fix protocol:**
1. Confirm `defvar FOO` exists in the source file.
2. Confirm the source file is `require`d **before** the test file is loaded.
3. If the variable is buffer-local, wrap the test in `with-temp-buffer`.
4. If defining a constant, prefer `defconst` to make intent explicit.

```elisp
;; Pattern for testing buffer-local state
(ert-deftest my-feature-test-buffer-local ()
  (with-temp-buffer
    (my-feature-mode 1)
    (should (boundp 'my-feature-local-var))))
```

---

#### Error Class B: `Symbol's function definition is void: FOO`

**Cause:** A function is called before its `defun` was evaluated, or the
providing file was not loaded.

**Fix protocol:**
1. Add `(require 'providing-feature)` at the top of the test file.
2. If the function is defined conditionally (e.g., inside `when`), ensure
   the condition is true in the test environment.
3. Use `fboundp` to write a defensive skip:

```elisp
(ert-deftest my-feature-test-optional-fn ()
  (skip-unless (fboundp 'my-optional-fn))
  (should (my-optional-fn)))
```

---

#### Error Class C: `Wrong type argument: TYPE, VALUE`

**Cause:** A function received an argument of unexpected type.
Common sub-cases:

| Emacs Error | Likely Root Cause |
|-------------|------------------|
| `Wrong type argument: listp, nil` | Missing inner `let` binding parens |
| `Wrong type argument: stringp, nil` | Variable not initialized; returns `nil` |
| `Wrong type argument: characterp, ...` | String passed where char expected |
| `Wrong type argument: number-or-marker-p` | `nil` used in arithmetic |

**Fix protocol:**
1. Isolate the failing expression; eval it independently in `*scratch*`.
2. Add a `(message "DEBUG: %S" var)` before the line to inspect runtime types.
3. Add a type guard at the function boundary:

```elisp
(defun my-fn (x)
  (cl-check-type x string)   ; signals error with clear message if wrong
  (upcase x))
```

---

#### Error Class D: `ERT test failed: (should EXPR)` — Logic Failure

**Cause:** The code runs but produces wrong output.

**Fix protocol:**
1. Print both expected and actual with `ert-fail`:

```elisp
(ert-deftest my-feature-test-value ()
  (let ((actual (my-feature-fn 5))
        (expected 10))
    (unless (equal actual expected)
      (ert-fail (format "Expected %S, got %S" expected actual)))))
```

2. Bisect: replace the full call with a known-good stub to confirm test
   harness works, then re-introduce real logic incrementally.

---

#### Error Class E: `File error: Cannot open load file`

**Cause:** `require` or `load` cannot find the file.

**Fix protocol — run in order, stop when fixed:**

```elisp
;; Step 1: Print current load-path
(message "%S" load-path)

;; Step 2: Confirm the file exists at expected location
(file-exists-p (expand-file-name "my-feature.el" my-pkg-dir))

;; Step 3: Confirm provide symbol matches
;; In my-feature.el the last line must be:
(provide 'my-feature)

;; Step 4: Add path and retry
(add-to-list 'load-path my-pkg-dir)
(require 'my-feature)
```

---

## 4. Miscellaneous Reliability Guardrails

### 4.1 Always Enable Lexical Binding

Every `.el` file the agent creates **must** have this as its first line:

```elisp
;;; filename.el --- Short description  -*- lexical-binding: t; -*-
```

Without `lexical-binding: t`, closures silently capture the wrong variables.
This is a non-negotiable default.

### 4.2 Use `cl-lib` Instead of Deprecated `cl`

```elisp
;; Bad
(require 'cl)
(defun* my-fn ...)

;; Good
(require 'cl-lib)
(cl-defun my-fn ...)
```

Use `cl-loop`, `cl-let`, `cl-flet`, `cl-labels`, `cl-destructuring-bind`,
`cl-check-type`, `cl-assert`, etc.

### 4.3 Byte-Compile to Surface Warnings Early

Before finalising any file, the agent should instruct the user to run:

```bash
emacs --batch --eval "(byte-compile-file \"my-feature.el\")"
```

Treat **every warning** as an error. Common warnings and their fixes:

| Warning | Fix |
|---------|-----|
| `reference to free variable` | Add `defvar` declaration or pass as argument |
| `function not defined` | Add `declare-function` or move `require` earlier |
| `obsolete function` | Replace with modern equivalent per warning message |
| `end of file during parsing` | Parenthesis mismatch — run audit from §1 |

### 4.4 `condition-case` Wrapping for Side-Effectful Operations

Any operation that touches the filesystem, network, or external processes
must be wrapped:

```elisp
(defun my-pkg-safe-read-file (path)
  "Return file contents as string or nil on any error."
  (condition-case err
      (with-temp-buffer
        (insert-file-contents (expand-file-name path))
        (buffer-string))
    (file-error
     (message "Could not read %s: %s" path (error-message-string err))
     nil)))
```

### 4.5 Autoload Cookies

For any function intended as a user-facing entry point, add the autoload
cookie immediately above `defun` so the package can be lazily loaded:

```elisp
;;;###autoload
(defun my-pkg-start ()
  "Entry point for my-pkg."
  (interactive)
  (my-pkg--internal-init))
```

### 4.6 Avoid `eval` at Runtime

`eval` bypasses byte-compilation, breaks lexical scope, and hides errors.
If dynamic dispatch is needed, use `funcall` or `apply` with explicit
function objects instead:

```elisp
;; Bad
(eval `(,fn ,arg))

;; Good
(funcall fn arg)
```

### 4.7 Interactive Testing Snippets

Use these patterns in `*scratch*` for rapid iteration before formalising
into ERT tests:

```elisp
;; Evaluate and print result without side effects in buffer
(pp (my-feature-fn test-input))

;; Time a call
(benchmark-run 100 (my-feature-fn test-input))

;; Inspect a macro expansion before trusting it
(pp (macroexpand-1 '(my-macro arg1 arg2)))
```

---

## Quick Reference Card

```
BEFORE WRITING CODE
  [ ] lexical-binding: t in file header
  [ ] cl-lib not cl

BEFORE OUTPUTTING A FORM
  [ ] Parenthesis Audit: open == close
  [ ] let bindings have double parens: ((x 0))
  [ ] Each top-level form on its own block

BEFORE REQUIRE
  [ ] load-path set with expand-file-name
  [ ] featurep guard if re-entrant context
  [ ] provide symbol matches filename

AFTER A TEST FAILURE
  [ ] Classify: void-var / void-fn / wrong-type / logic / load-error
  [ ] Follow the matching Self-Correction protocol from §3.3
  [ ] Add type guard or skip-unless if environment-dependent

BEFORE SHIPPING
  [ ] byte-compile with --batch, zero warnings
  [ ] ert-run-tests-batch-and-exit green
  [ ] No runtime eval calls
```
