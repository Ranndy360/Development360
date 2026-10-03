# Senior checklist — React / Next.js (App Router)

Read once in preflight; walk it in the verify step for every file you touched. The project's
`CLAUDE.md`, ESLint config and existing precedents win over anything here.

## Architecture & component design

- **Server Components by default.** Add `'use client'` only to the smallest leaf that needs
  state, effects, browser APIs or event handlers. Never mark a page or layout as client
  because one button inside needs `onClick` — extract the button.
- **Fetch on the server.** Pages and Server Components fetch directly (`await`), in parallel
  (`Promise.all`) when independent — no request waterfalls. Client components receive data
  as props or via the project's server-state library; they do not fetch what the server
  could have fetched.
- **Mutations go through Server Actions or route handlers**, validated with the project's
  schema library (zod/valibot) on the server — never trust client validation alone. After a
  mutation, revalidate (`revalidatePath` / `revalidateTag` / `updateTag`) rather than
  refetching manually.
- **Route conventions**: every new route segment that fetches gets `loading.tsx` (or a
  `Suspense` boundary) and `error.tsx`; `not-found.tsx` where a lookup can miss; `metadata`
  / `generateMetadata` for anything user-facing.
- **Colocate.** Feature-private components, hooks and tests live beside the route
  (`_components/`, `_hooks/`); only truly shared things go to `components/`, `hooks/`,
  `lib/`. Follow the nearest precedent.
- **Composition over configuration.** Prefer `children` and slots to boolean prop
  explosions. One component, one responsibility; split when a file mixes data loading,
  layout and interaction.
- **Hooks**: a custom hook for any logic reused twice or testable on its own. Respect the
  rules of hooks; no hooks inside conditions. Derive state instead of syncing it with
  `useEffect`; reach for `useEffect` only for real external synchronisation.
- **State lives as low as possible**: URL (search params) for shareable UI state, server
  state in the server-state library, local `useState` for ephemeral UI, global store only
  for truly global concerns already managed by the project's store.

## TypeScript

- Strict mode is assumed. No `any`, no non-null assertions to silence errors, no `as`
  casts that hide a real mismatch. Narrow with type guards.
- Props typed with an exported `interface`/`type` named `<Component>Props`. Prefer
  discriminated unions over optional-everything props.
- Types for fetched data come from the API client, generated types, or a schema (`z.infer`)
  — never hand-written duplicates of a backend DTO when a source of truth exists.
- Env vars are read through the project's typed env module if there is one; only
  `NEXT_PUBLIC_*` variables may reach client code.

## Accessibility (non-negotiable)

- Semantic HTML first (`button`, `nav`, `main`, `ul`, `table`, headings in order). ARIA only
  when no native element does the job.
- Every interactive element is keyboard-reachable and operable, has a visible focus style,
  and an accessible name (visible label, `aria-label`, or `aria-labelledby`).
- Form fields have associated `<label>`s; errors are announced (`aria-describedby`,
  `aria-invalid`, live region for async errors).
- Images have meaningful `alt` or `alt=""` when decorative. Icon-only buttons have a label.
- Colour is never the only carrier of meaning; contrast meets WCAG AA.
- Dialogs/menus trap focus, close on `Escape`, and return focus — use the project's UI
  primitives (shadcn/Radix/Headless UI) instead of hand-rolling.
- Run the project's a11y lint (`eslint-plugin-jsx-a11y`) and, in RTL tests, query by role —
  if you cannot query it by role, it is probably not accessible.

## Performance

- Use `next/image`, `next/link`, `next/font`; no raw `<img>` for content images, no
  `<a href>` for internal navigation.
- Keep the client bundle small: heavy or rarely used client components load via
  `next/dynamic`; no server-only libraries imported into client files (`server-only`
  package where the project uses it).
- Stream with `Suspense` around slow data instead of blocking the whole page.
- Lists: stable `key`s (ids, never index when items reorder). Memoise (`memo`, `useMemo`,
  `useCallback`) only when a measurement or an obvious hot path justifies it — not by
  default. If the project uses the React Compiler, do not add manual memoisation.
- Be explicit about caching and rendering mode for every new route (`dynamic`,
  `revalidate`, `cacheLife`/`cacheTag` where the project uses Cache Components). Do not
  let a page silently become fully dynamic because of one uncached call.
- No layout shift: reserve space for images, skeletons match final layout.

## UX states & error handling

- Every data-driven view handles **loading**, **empty**, **error** and **success**. Empty
  states say what to do next. Errors are human-readable, never raw stack text.
- Forms: disable the submit while pending (`useFormStatus` / pending flag), keep the user's
  input on failure, show field-level errors next to the field and a summary when needed,
  announce success.
- Optimistic UI only where the project already does it and rollback is implemented.
- Unexpected errors reach the project's error reporting (Sentry etc.) via `error.tsx` /
  the global error boundary, not `console.error` alone.

## Security

- Authorization is enforced in the server code that reads or mutates data (Server Action,
  route handler, data layer) — middleware/proxy is a convenience layer, not the guard.
- Never ship secrets, tokens or internal URLs to the client. Grep for the variable name in
  client files before you finish.
- Sanitise anything rendered with `dangerouslySetInnerHTML`; avoid it when possible.
- Server Actions: validate input, check the caller's identity, return typed results —
  never throw raw DB errors to the client.
- External links: `rel="noopener noreferrer"` when `target="_blank"`.

## Testing bar

- Unit/component: React Testing Library + `userEvent`; assert what the user sees. Cover
  the happy path, each validation/error branch, the empty state, and keyboard interaction
  for custom widgets.
- Hooks: `renderHook`; test the contract, not the internals.
- Server Actions / route handlers / data functions: plain function tests with the data
  layer mocked at its boundary; one test per authorization/validation branch.
- Network: MSW handlers shared in the test setup; no ad-hoc `jest.fn()` on `fetch`.
- E2E (Playwright/Cypress): only the critical journey named in the spec; role-based
  selectors; no sleeps — wait on state.
- No tests that pass regardless of the implementation (assert on outcomes, not calls, unless
  the call *is* the outcome).

## Documentation

- JSDoc on exported components, hooks and actions: what it does, non-obvious props,
  examples when usage is not self-evident. Do not document the obvious.
- New env variable → `.env.example` + the project's env docs.
- New route / Server Action / public component → add to whatever inventory the project
  keeps (README section, `docs/`, Storybook story, ADR). If there is a Storybook, a story
  per visual state of a new shared component.
- Decisions that deviate from the spec or from the obvious approach go in the spec's
  **Agent summary → Key decisions**, with the reason.

## Before you call the spec done

- [ ] `typecheck`, `lint`, full `test`, `build` green (and `test:e2e` when required)
- [ ] No new `'use client'` above the leaf that needs it
- [ ] Loading / empty / error / success states present and tested
- [ ] Keyboard + screen-reader path works for anything you added
- [ ] No secrets or server-only imports in client files
- [ ] Docs, stories, `.env.example` updated; spec and order file recorded
