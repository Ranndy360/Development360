---
name: implement-specs-react-native
description: Implement the specs of a React Native (Expo or bare) project one by one, following an implementation-order file produced by /us-to-specs. Screen-by-screen tracer bullets, TDD with React Native Testing Library, accessibility, offline and performance checks, typecheck/lint/test feedback loop, one commit per spec, resumable.
argument-hint: "[IMPLEMENTATION_ORDER_FILE] [--spec NN | --from NN] [--no-branch] [--no-commit] [--e2e] [--platform ios|android]"
disable-model-invocation: true
---

IMPLEMENTATION_ORDER_FILE: $ARGUMENTS[0]
OPTIONS: $ARGUMENTS[1..] (optional)

- `--spec NN` — implement only spec `NN`, then stop.
- `--from NN` — start at spec `NN` (skips earlier ones even if not marked completed).
- `--no-branch` — stay on the current branch.
- `--no-commit` — leave changes uncommitted (the developer will commit).
- `--e2e` — also run the end-to-end suite (Maestro/Detox) in the per-spec verify step.
  Without it, e2e runs only when a spec explicitly asks for it.
- `--platform ios|android` — when a native check is needed, target only this platform.

# Preflight (once)

1. Read the implementation-order file. It has four sections: the ordered table, **Completed
   specs**, **Recommended Implementation Order**, **Notes**. Specs listed under Completed are
   skipped — this is how a run resumes.
2. Read the project's `CLAUDE.md` and any path-scoped rules for the screens/features the
   specs touch. They override anything below.
3. Detect the stack and toolchain. Record what you find before editing anything:
   - **Runtime**: Expo (managed / dev client / prebuild) or bare React Native. Read
     `app.json` / `app.config.ts`, `package.json`, and whether `ios/` and `android/` are
     committed. New Architecture on or off (`newArchEnabled`).
   - **Navigation**: Expo Router (file-based, `app/`) or React Navigation (`navigation/`
     with typed param lists). Follow whichever is in place; never mix.
   - **Scripts** in `package.json`: `test`, `lint`, `typecheck` (or `tsc --noEmit`),
     `test:e2e`, `format`, and any native check the project uses as its "build" gate
     (`expo export`, `expo prebuild --no-install`, `expo-doctor`, a Gradle/Xcode script).
     Native builds (EAS, Xcode, Gradle) are slow and are **not** part of the per-task loop —
     they run once per spec only if the project's rules say so, otherwise they are left to
     CI and noted in the summary.
   - **Test runner**: Jest with `jest-expo` or `react-native` preset; `@testing-library/
     react-native`; its `userEvent`; MSW (or the project's mock server); the jest setup file
     that mocks native modules. E2E: Maestro (`.maestro/`) or Detox. Note how a single test
     file is run.
   - **Conventions already in the repo**: styling (StyleSheet + theme tokens, NativeWind,
     Tamagui, styled-components), UI kit, forms (react-hook-form + zod), server state
     (TanStack Query), client state (Zustand, Redux Toolkit), persistence (MMKV,
     AsyncStorage, SecureStore), lists (`FlatList`, `FlashList`), animations (Reanimated),
     images (`expo-image`), i18n, analytics, crash reporting. Reuse them; do not add a
     competing library.
   - **Folder layout**: feature folders vs `screens/`, `components/`, `hooks/`,
     `services/`. Mirror the nearest precedent.
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
MCP server or the image is available, look at it before planning — note safe areas, keyboard
behaviour and the states (empty/loading/error) the design shows or omits.

## 2. Break down

Split the spec into the smallest tasks that each leave typecheck and tests green. Order them:

1. critical fixes the spec calls out,
2. the **tracer bullet** — the thinnest end-to-end path (route/screen registered →
   navigation reachable → data fetched → one rendered element → one test), with loading
   and error states stubbed but present,
3. the remaining acceptance criteria, one per task — each usually maps to a screen state
   (empty, loading, error, success, validation, offline, permission denied) or a gesture,
4. polish: accessibility pass, platform pass (iOS and Android quirks, safe areas,
   keyboard), performance pass (lists, re-renders, images), docs.

If a task turns out to need a refactor first (a screen that must be split, a hook extracted
from a component, a native module wrapper), say **"HANG ON A SECOND"**, carve out the
refactor as its own task, finish it with its tests, then return.

## 3. Explore

Read the files the spec names, the precedent it mirrors (a sibling screen or component),
the shared primitives you will reuse (theme, UI kit, API client, query hooks), the
navigation types, and their existing tests. Fill your context before editing; do not guess
at route params, API response shapes or native module APIs — open the type or the docs.

## 4. Execute with TDD

For each task: write or extend the failing test → make it pass → tidy. Stay inside the
spec's scope — no extra features, no unrelated cleanup, no new abstractions the spec does
not need.

Testing rules for this stack:

- **Screens and components**: React Native Testing Library. Query by role, label or text
  as a user would (`getByRole`, `getByLabelText`, `getByText`), interact with `userEvent`
  (press, type, scroll), assert on what the user sees. Never assert on style objects,
  internal state or implementation details. No snapshot tests unless the project already
  relies on them.
- **Screens are rendered inside the project's test wrapper** (navigation container or
  Expo Router test helpers, query client, theme, safe-area provider). If no wrapper exists,
  create one in the test utils as its own task.
- **Hooks, services, stores**: `renderHook` for hooks; plain function tests for API
  clients, stores and utils, with the network mocked by MSW handlers and native modules
  mocked in the jest setup — never ad hoc `jest.mock` of `fetch` inside one test.
- **Native modules** (camera, location, notifications, secure store): wrap them in a thin
  module the app imports, mock *that* module in tests, and test the permission-denied and
  unavailable paths explicitly.
- **Timers and animations**: use fake timers for debounce/throttle; do not test Reanimated
  internals — test the resulting state.
- **E2E**: only for the critical user journey the spec names, and only when `--e2e` is set
  or the spec asks for it. Maestro flows / Detox tests use `testID`s per project
  convention; no sleeps, wait on elements.
- Every changed screen/component/hook/service gets its sibling test updated in the same
  task. Test files live where the project puts them (`*.test.tsx` next to the file, or
  `__tests__/`).

Fast feedback while iterating: run only the related test files (`jest <path>` /
`--findRelatedTests`) plus `typecheck`. Do not run the whole suite after every edit. Do not
start a simulator or a native build inside the task loop.

## 5. Verify the spec

Run, in order, and fix anything red before moving on:

1. `typecheck` (or `tsc --noEmit`)
2. `lint` (ESLint with the React Native / Expo config; `format --check` if present)
3. the **full** `test` suite
4. the project's bundle/native sanity gate if it has one (`expo export`, `expo-doctor`,
   `expo prebuild --no-install`, or a project script) — it catches bad imports of native
   modules, config plugin errors and asset problems that unit tests never see. If the
   project has no such gate, skip and say so in the summary.
5. `test:e2e` if `--e2e` or the spec requires it (`--platform` narrows the target).
6. Walk the **senior checklist** (`references/senior-checklist.md`) for the files you
   touched: a11y, safe area and keyboard, states incl. offline, list performance,
   permissions, secrets. Fix what fails; note in the spec what you consciously deferred and
   why.

## 6. Record

- Tick the spec's acceptance criteria.
- Append to **Log of progress**: `- <YYYY-MM-DD>: Completed`.
- Append an **Agent summary** section: *Work completed*, *Key decisions* (e.g. "used
  `FlashList` because the list is unbounded; kept `FlatList` elsewhere"), *Files changed*
  (paths only), *Manual QA* (what to tap on a device to see it, per platform if it
  differs). Keep it short.
- Move the spec into **Completed specs** in the implementation-order file with a one-line
  summary of what landed and any decision that deviates from the spec.
- Update documentation the project keeps for what you touched: component/hook JSDoc, the
  feature's README or `docs/` page, Storybook stories if the project uses Storybook, the
  deep-link / route inventory, `.env.example` and `app.config.ts` docs for new variables,
  permission strings (`Info.plist` usage descriptions, Android manifest) and their
  rationale, store-listing notes when a new permission is introduced.

## 7. Commit (unless `--no-commit`)

`git add` the specific files you changed (review `git status` first — never add secrets,
`.env*` files, keystores, provisioning profiles, or unrelated generated native files), then
commit with the project convention, e.g. `<TICKET> <description>`. Respect hooks; never use
`--no-verify`. If a hook fails, fix the cause and commit again as a **new** commit. Use `gh`
only for pull requests, not for commits.

## 8. Next

Continue with the next spec. If `--spec NN` was given, stop here.

# Blockers

If something outside the spec blocks you (missing design decision, an API contract the
backend has not shipped, a native module that needs a new dev client build, a credential
only the developer has), append `- <YYYY-MM-DD>: Blocked — <concise reason>` to the spec's
log, add the same line to the order file's **Notes**, and stop. Do not work around it with
destructive or hook-bypassing actions, and do not ship a mocked API or native module as if
it were real — if you must stub, say so in the spec and leave a `TODO(<TICKET>)` at the stub.

# Finish

When every spec in the table is under **Completed specs**, print a short summary (branch,
commits, new env vars or permissions, whether a new native build / dev client is required,
anything the developer must do by hand such as adding a secret to EAS or updating store
metadata) and output:

`<promise>COMPLETE</promise>`

Suggest opening the pull request with `gh pr create` only — do not create it yourself unless
asked.

# Final rules

- One spec at a time. Finish, verify, record, commit — then the next.
- Scope is the spec. Anything else goes to a note, not into the diff.
- Every screen state (loading, empty, error, offline, success) exists and is tested before
  polish; every tappable thing has an accessible name.
- Native builds are a gate for CI, not for the edit loop.
- The project's `CLAUDE.md` wins over this file whenever they disagree.
