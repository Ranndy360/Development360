---
name: implement-specs-nestjs
description: Implement the specs of a NestJS project one by one, following an implementation-order file produced by /us-to-specs. TDD, build/test/lint feedback loop, one commit per spec, resumable.
argument-hint: "[IMPLEMENTATION_ORDER_FILE] [--spec NN | --from NN] [--no-branch] [--no-commit]"
disable-model-invocation: true
---

IMPLEMENTATION_ORDER_FILE: $ARGUMENTS[0]
OPTIONS: $ARGUMENTS[1..] (optional)

- `--spec NN` — implement only spec `NN`, then stop.
- `--from NN` — start at spec `NN` (skips earlier ones even if not marked completed).
- `--no-branch` — stay on the current branch.
- `--no-commit` — leave changes uncommitted (the developer will commit).

# Preflight (once)

1. Read the implementation-order file. It has four sections: the ordered table, **Completed
   specs**, **Recommended Implementation Order**, **Notes**. Specs listed under Completed are
   skipped — this is how a run resumes.
2. Read the project's `CLAUDE.md` and any path-scoped rules for the modules the specs touch.
   They override anything below.
3. Detect the toolchain from `package.json` scripts: `build`, `test`, `lint`, and a
   narrower test script if one exists (`test:related`, `test:affected`, `test:changed`).
   Note how a single spec file is run (e.g. `jest <path>`), and whether tests need an env
   file. Prefer the project's own scripts over raw binaries.
4. Derive the ticket id and title from the specs folder name (`<TICKET>-<kebab-title>`).
   Read the commit convention from `CLAUDE.md` / recent `git log`; default to
   `<TICKET> <description>`.
5. Unless `--no-branch`: run `git status`; if the tree is dirty, stop and ask — never stash or
   discard someone's work silently. Then create and switch to `feat/<TICKET>-<kebab-title>`
   from the current HEAD (reuse it if it already exists).

# Per spec — ONE AT A TIME, in table order

## 1. Load

Read the spec file. Confirm its **Blocked by** specs are completed; if not, stop and say
which one is missing.

## 2. Break down

Split the spec into the smallest tasks that each leave the build green. Order them:

1. critical fixes the spec calls out,
2. the **tracer bullet** — the thinnest end-to-end path (one field, one case, one test),
3. the remaining acceptance criteria, one per task,
4. polish and refactors.

If a task turns out to need a refactor first, say **"HANG ON A SECOND"**, carve out the
refactor as its own task, finish it, then return.

## 3. Explore

Read the files the spec names plus their specs and the precedent it mirrors. Fill your
context before editing; do not guess at signatures.

## 4. Execute with TDD

For each task: write or extend the failing test → make it pass → tidy. Stay inside the
spec's scope — no extra features, no unrelated cleanup, no new abstractions the spec does
not need. Every changed `*.service.ts` / `*.controller.ts` / util gets its sibling spec
updated in the same task.

Fast feedback while iterating: run only the related spec files. Do not run the whole suite
after every edit.

## 5. Verify the spec

Run, in order, and fix anything red before moving on:

1. `build`
2. the **full** `test` suite
3. `lint`

## 6. Record

- Tick the spec's acceptance criteria.
- Append to **Log of progress**: `- <YYYY-MM-DD>: Completed`.
- Append an **Agent summary** section: *Work completed*, *Key decisions*, *Files changed*
  (paths only). Keep it short.
- Move the spec into **Completed specs** in the implementation-order file with a one-line
  summary of what landed and any decision that deviates from the spec.
- Update project docs/rules the project requires for the modules touched.

## 7. Commit (unless `--no-commit`)

`git add` the specific files you changed (review `git status` first — never add secrets or
unrelated files), then commit with the project convention, e.g. `<TICKET> <description>`.
Respect hooks; never use `--no-verify`. If a hook fails, fix the cause and commit again as a
**new** commit. Use `gh` only for pull requests, not for commits.

## 8. Next

Continue with the next spec. If `--spec NN` was given, stop here.

# Blockers

If something outside the spec blocks you (missing decision, failing migration, contradictory
rule), append `- <YYYY-MM-DD>: Blocked — <concise reason>` to the spec's log, add the same
line to the order file's **Notes**, and stop. Do not work around it with destructive or
hook-bypassing actions.

# Finish

When every spec in the table is under **Completed specs**, print a short summary (branch,
commits, anything the developer must do by hand such as applying a migration) and output:

`<promise>COMPLETE</promise>`

Suggest opening the pull request with `gh pr create` only — do not create it yourself unless
asked.

# Final rules

- One spec at a time. Finish, verify, record, commit — then the next.
- Scope is the spec. Anything else goes to a note, not into the diff.
- The project's `CLAUDE.md` wins over this file whenever they disagree.
