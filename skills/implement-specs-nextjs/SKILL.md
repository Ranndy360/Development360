---
name: implement-specs-nextjs
description: Implement the specs of a React / Next.js (App Router) project one by one, following an implementation-order file produced by /us-to-specs. Server-first components, TDD with React Testing Library, accessibility and performance checks, typecheck/build/test/lint feedback loop, one commit per spec, resumable.
argument-hint: "[IMPLEMENTATION_ORDER_FILE] [--spec NN | --from NN] [--no-branch] [--no-commit] [--e2e]"
disable-model-invocation: true
---

IMPLEMENTATION_ORDER_FILE: $ARGUMENTS[0]
OPTIONS: $ARGUMENTS[1..] (optional)

- `--spec NN` — implement only spec `NN`, then stop.
- `--from NN` — start at spec `NN` (skips earlier ones even if not marked completed).
- `--no-branch` — stay on the current branch.
- `--no-commit` — leave changes uncommitted (the developer will commit).
- `--e2e` — also run the end-to-end suite (Playwright/Cypress) in the per-spec verify step.
  Without it, e2e runs only when a spec explicitly asks for it.

# Preflight (once)

1. Read the implementation-order file. It has four sections: the ordered table, **Completed
   specs**, **Recommended Implementation Order**, **Notes**. Specs listed under Completed are
   skipped — this is how a run resumes.
2. Read the project's `CLAUDE.md` and any path-scoped rules for the routes/features the specs
   touch. They override anything below.
3. Detect the stack and toolchain. Record what you find before editing anything:
   - **Router**: `app/` (App Router, default assumption) or `pages/`. Never mix patterns in
     one spec; follow what the touched route already uses.
   - **Scripts** in `package.json`: `build`, `test`, `lint`, `typecheck` (or `tsc --noEmit`),
     `test:e2e`, `format`, `storybook`. Prefer the project's scripts over raw binaries.
   - **Test runner**: Vitest or Jest; React Testing Library; `@testing-library/user-event`;
     MSW for network mocking; Playwright or Cypress for e2e. Note how a single test file is
     run (e.g. `vitest run <path>`) and whether tests need an env file.
   - **Conventions already in the repo**: styling (Tailwind, CSS Modules, styled-components),
     UI kit (shadcn/ui, MUI…), forms (react-hook-form + zod, Server Actions +
     `useActionState`), server state (TanStack Query, plain `fetch`), client state (Zustand,
     Context), i18n, analytics, auth helper, error reporting. Reuse them; do not add a
     competing library.
   - **Folder layout**: colocated features (`app/(group)/feature/_components`) vs shared
     `components/`, `lib/`, `hooks/`. Mirror the nearest precedent.
4. Derive the ticket id and title from the specs folder name (`<TICKET>-<kebab-title>`).
   Read the commit convention from `CLAUDE.md` / recent `git log`; default to
   `<TICKET> <description>`.
5. Unless `--no-branch`: run `git status`; if the tree is dirty, stop and ask — never stash or
   discard someone's work silently. Then create and switch to `feat/<TICKET>-<kebab-title>`
   from the current HEAD (reuse it if it already exists).
6. Read `references/senior-checklist.md` (next to this file) once. It is the quality bar for
   every task below; the project's own rules win where they disagree.

# Per spec — ONE AT A TIME, in table order

## 1. Load

Read the spec file. Confirm its **Blocked by** specs are completed; if not, stop and say
which one is missing. If the spec references a design (Figma link, screenshot) and a design
MCP server or the image is available, look at it before planning.

## 2. Break down

Split the spec into the smallest tasks that each leave typecheck and tests green. Order them:

1. critical fixes the spec calls out,
2. the **tracer bullet** — the thinnest end-to-end path (route → data → one rendered
   element → one test), with loading and error states stubbed but present,
3. the remaining acceptance criteria, one per task — each one usually maps to a state of the
   UI: empty, loading, error, success, validation, permissions,
4. polish: accessibility pass, responsive pass, performance pass, docs.

If a task turns out to need a refactor first (a component that must be split, a hook that
must be extracted), say **"HANG ON A SECOND"**, carve out the refactor as its own task,
finish it with its tests, then return.

## 3. Explore

Read the files the spec names, the precedent it mirrors (a sibling route or component), the
shared primitives you will reuse (`components/ui`, `lib/`), and their existing tests. Fill
your context before editing; do not guess at props, server action signatures, or the shape
of fetched data — open the type.

## 4. Execute with TDD

For each task: write or extend the failing test → make it pass → tidy. Stay inside the
spec's scope — no extra features, no unrelated cleanup, no new abstractions the spec does
not need.

Testing rules for this stack:

- **Components and hooks**: React Testing Library. Query by role/label/text as a user would
  (`getByRole`, `getByLabelText`), interact with `userEvent`, assert on what the user sees.
  Never assert on class names, internal state, or implementation details. No snapshot tests
  unless the project already relies on them.
- **Server Components, Server Actions, route handlers, loaders**: test them as plain
  functions with their inputs mocked at the boundary (DB client, external API via MSW).
  Validation and authorization paths get their own test cases.
- **Network**: mock with MSW handlers, not by mocking `fetch` ad hoc.
- **E2E**: only for the critical user journey the spec names, and only when `--e2e` is set
  or the spec asks for it. Keep selectors role-based or `data-testid` per project convention.
- Every changed component/hook/action/util gets its sibling test updated in the same task.
  Test files live where the project puts them (`*.test.tsx` next to the file, or
  `__tests__/`).

Fast feedback while iterating: run only the related test files (`vitest run <path>` /
`jest <path>` / `--findRelatedTests`) plus `typecheck`. Do not run the whole suite after
every edit.

## 5. Verify the spec

Run, in order, and fix anything red before moving on:

1. `typecheck` (or `tsc --noEmit`)
2. `lint` (ESLint with the Next.js config; `format --check` if the project has it)
3. the **full** `test` suite
4. `build` (`next build`) — this catches server/client boundary errors, invalid metadata,
   dynamic/static conflicts and missing env vars that tests never see
5. `test:e2e` if `--e2e` or the spec requires it
6. Walk the **senior checklist** (`references/senior-checklist.md`) for the files you
   touched: server/client boundary, a11y, loading/error/empty states, performance, security.
   Fix what fails; note in the spec what you consciously deferred and why.

## 6. Record

- Tick the spec's acceptance criteria.
- Append to **Log of progress**: `- <YYYY-MM-DD>: Completed`.
- Append an **Agent summary** section: *Work completed*, *Key decisions* (e.g. "kept the
  table a Server Component; only the filter bar is `'use client'`"), *Files changed*
  (paths only). Keep it short.
- Move the spec into **Completed specs** in the implementation-order file with a one-line
  summary of what landed and any decision that deviates from the spec.
- Update documentation the project keeps for what you touched: component JSDoc, the
  feature's README or `docs/` page, Storybook stories if the project uses Storybook, env
  variable docs (`.env.example`) for any new variable, route or Server Action inventory if
  one exists.

## 7. Commit (unless `--no-commit`)

`git add` the specific files you changed (review `git status` first — never add secrets,
`.env*` files, or unrelated files), then commit with the project convention, e.g.
`<TICKET> <description>`. Respect hooks; never use `--no-verify`. If a hook fails, fix the
cause and commit again as a **new** commit. Use `gh` only for pull requests, not for commits.

## 8. Next

Continue with the next spec. If `--spec NN` was given, stop here.

# Blockers

If something outside the spec blocks you (missing design decision, an API contract the
backend has not shipped, a flaky e2e that is not yours), append
`- <YYYY-MM-DD>: Blocked — <concise reason>` to the spec's log, add the same line to the
order file's **Notes**, and stop. Do not work around it with destructive or hook-bypassing
actions, and do not ship a mocked API as if it were the real one — if you must stub, say so
in the spec and leave a `TODO(<TICKET>)` at the stub.

# Finish

When every spec in the table is under **Completed specs**, print a short summary (branch,
commits, new env vars, anything the developer must do by hand such as adding a secret in
Vercel or running a migration) and output:

`<promise>COMPLETE</promise>`

Suggest opening the pull request with `gh pr create` only — do not create it yourself unless
asked.

# Final rules

- One spec at a time. Finish, verify, record, commit — then the next.
- Scope is the spec. Anything else goes to a note, not into the diff.
- Server Component by default; `'use client'` only on the leaf that needs it.
- Every screen state (loading, empty, error, success) exists and is tested before polish.
- The project's `CLAUDE.md` wins over this file whenever they disagree.
