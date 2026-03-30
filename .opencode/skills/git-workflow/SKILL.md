---
name: git-workflow
description: Git conventions for the lightweight workflow. Covers branch naming, squash merge sequence, conventional commit format, phase tagging, and the regression gate that runs on main after every merge. Used by the merger subagent.
license: MIT
compatibility: opencode
---

## Branch naming

Feature branches are created at the start of each phase by the build agent.

Format: `phase/<N>-<slug>`
- `N` — phase number from SPEC.md (1-indexed)
- `slug` — 2–3 word kebab-case summary of the phase goal

Examples:
phase/1-scaffold phase/2-backend-api phase/3-frontend-ui
Never create branches named `fix/`, `hotfix/`, or `chore/` inside this workflow — all work happens on phase branches. If a regression is found on main after merge, the fix goes on a new phase branch (`phase/<N+1>-regression-fix`) with a single task in SPEC.md.

---

## Pre-merge checklist

The merger runs this checklist before touching `main`. If any item fails, abort.
bash # 1. Confirm you are on the phase branch git branch --show-current # Expected: phase/<N>-<slug> # 2. Run check one final time on the phase branch make check # Expected: { "status": "pass" } # 3. Confirm no untracked or uncommitted files git status # Expected: nothing to commit, working tree clean # 4. Confirm the branch is ahead of main (not behind) git log main..HEAD --oneline # Expected: at least one commit
If `make check` fails at this step, return to the reviewer. Do not proceed.

---

## Squash merge sequence
bash # Switch to main git switch main # Pull any remote changes (if working with a remote) # git pull origin main # Squash merge the phase branch git merge --squash phase/<N>-<slug> # Commit with conventional message (see format below) git commit -m "feat(phase-<N>): <phase goal>" # Tag the commit git tag phase-<N> # Push (if working with a remote) # git push origin main --tags
**Never use `git merge` without `--squash`.** A merge commit on main creates noise in the log and makes it harder to bisect regressions. The full development history lives on the phase branch; main's history is one commit per phase.

---

## Conventional commit format

The commit message is derived from SPEC.md's phase heading, lowercased and written in imperative mood.

Format: `feat(phase-<N>): <imperative phrase>`

| Phase heading in SPEC.md | Commit message |
|---|---|
| `Phase 1 — Scaffold` | `feat(phase-1): scaffold project structure and dependencies` |
| `Phase 2 — Backend API` | `feat(phase-2): implement backend API with SQLite persistence` |
| `Phase 3 — Frontend UI` | `feat(phase-3): build next.js frontend with filter and todo operations` |

Rules:
- Lowercase after the colon.
- Imperative: "implement", "build", "add", "wire up" — not "implemented" or "implementation of".
- No period at the end.
- Max 72 characters total.

---

## Phase tag format

Tags are lightweight (not annotated) and applied immediately after the merge commit.
bash git tag phase-1 # after phase 1 merges git tag phase-2 # after phase 2 merges
Tags provide a stable reference point for bisecting regressions: if a regression is found after `phase-3` merges, `git diff phase-2 phase-3` shows exactly what changed.

---

## Regression gate

After every merge to main, the full test suite runs on `main`. This is non-negotiable.
bash git switch main make check
**If the regression gate fails on main:**
1. Do not proceed to the next phase.
2. Do not attempt a quick fix directly on `main`.
3. Create a divergence entry in SPEC.md:
## DIVERGENCE — regression — <date> **Phase that introduced regression:** phase-<N> **Failing test:** <test name> **Proposed options:** ...
4. Wait for human decision on whether to revert the merge or fix forward on a new phase branch.

A red main is the highest-priority issue in the system. It blocks everything.

---

## SPEC.md bookkeeping after merge

After a successful merge and green regression gate, the merger makes exactly two changes to SPEC.md:

1. Mark all tasks in the completed phase as `[x]`:
- [x] P2-T1 Todo SQLModel + DB init [tdd] - [x] P2-T2 POST /todos endpoint [tdd]
2. Remove the `## Current phase` section if present.

No other sections are touched. The merger does not rewrite the architecture, amend contracts, or add commentary.

---

## Identifying the next unblocked phase

After SPEC.md is updated, scan the task list for the next phase where:
- At least one task is `[ ]` (incomplete)
- No task has a `blocked_by:` annotation pointing to an incomplete task from a prior phase

Report the phase name and its task list. If no such phase exists, all phases are complete — report a project summary instead.
