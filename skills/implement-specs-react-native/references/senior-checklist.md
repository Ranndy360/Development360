# Senior checklist — React Native (Expo / bare)

Read once in preflight; walk it in the verify step for every file you touched. The project's
`CLAUDE.md`, ESLint config and existing precedents win over anything here.

## Architecture & component design

- **Screens are thin.** A screen composes a data hook (query/mutation), a presentational
  component tree and navigation glue. Business logic lives in hooks/services that can be
  tested without rendering.
- **Navigation is typed.** Expo Router: typed routes enabled and `href`s typed; React
  Navigation: every stack/tab has a `ParamList` and screens use the typed `useNavigation` /
  `useRoute`. New screens are registered where the precedent registers them, with a deep
  link if the project maintains a linking config.
- **State lives as low as possible**: server state in the server-state library (with
  stale/retry/offline settings matching the project's defaults), ephemeral UI state in
  `useState`, global store only for truly global concerns. Persist only what must survive
  a restart, through the project's persistence layer (MMKV/AsyncStorage); secrets through
  `expo-secure-store` / Keychain, never AsyncStorage.
- **Native modules are wrapped** in a small app-owned module (permissions, camera,
  location, notifications, biometrics). The app imports the wrapper; tests mock the wrapper.
- **Composition over configuration**: `children`/slots over boolean prop explosions; one
  component, one responsibility. Hooks for logic reused twice or testable on its own.
- Derive state instead of syncing it with `useEffect`; `useEffect` is for real external
  synchronisation (subscriptions, app-state, listeners) and always cleans up.
- Follow the project's folder precedent (feature folders vs `screens/`/`components/`).

## TypeScript

- Strict mode assumed. No `any`, no non-null assertions to silence errors, no `as` casts
  hiding a real mismatch. Narrow with type guards.
- Props typed with an exported `<Component>Props`. Discriminated unions over
  optional-everything props.
- API types come from the client, generated types or a schema (`z.infer`) — never
  hand-written copies of a backend DTO when a source of truth exists.
- Env/config is read through `app.config.ts` `extra` or the project's typed config module;
  nothing secret goes into the JS bundle — it is shipped to every user.

## Platform & layout

- Safe areas via `react-native-safe-area-context` (or the router's defaults) on every new
  screen; nothing under the notch, home indicator or status bar.
- Keyboard: inputs stay visible (`KeyboardAvoidingView` with the project's behaviour per
  platform, or the project's keyboard-aware list/scroll); "return" key type and
  `autoCapitalize`/`autoComplete`/`textContentType` set sensibly; tapping outside dismisses
  where the design expects it.
- `Platform.select` / `Platform.OS` only for real differences; shared code everywhere else.
  Check both platforms' behaviour for anything touching navigation bars, modals, date
  pickers, shadows/elevation, haptics, fonts.
- Use theme tokens (spacing, colours, typography) from the project's theme; no magic
  numbers or hard-coded colours. Respect dark mode if the project supports it.
- Responsive to font scaling and small/large screens: no fixed heights that clip scaled
  text; `allowFontScaling` stays on unless the project decided otherwise.
- Status bar style set per screen when the background changes.

## Accessibility (non-negotiable)

- Every tappable element has `accessibilityRole` and an accessible name
  (`accessibilityLabel` or visible text); icon-only buttons always get a label.
- `accessibilityState` for selected/disabled/checked/expanded; `accessibilityHint` when the
  outcome is not obvious; `accessibilityValue` for sliders/progress.
- Touch targets ≥ 44×44 pt (`hitSlop` when the visual is smaller).
- Group related content (`accessible` + a combined label) so screen readers do not read
  each fragment separately; decorative images are hidden from the accessibility tree.
- Announce async results (`AccessibilityInfo.announceForAccessibility` or a live region)
  for toasts/errors; focus moves to new modals/sheets and returns on close.
- Colour is never the only carrier of meaning; contrast meets WCAG AA.
- In RNTL tests query by role/label — if you cannot, it is probably not accessible.

## Performance

- Lists: `FlatList`/`FlashList` (never `ScrollView` + `map` for unbounded data), stable
  `keyExtractor` (ids, never index), `renderItem` as a memoised component, `getItemType` /
  `estimatedItemSize` for FlashList, pagination through the project's query hook.
- Images through `expo-image` (or the project's image component) with explicit sizes,
  caching policy and placeholders; no full-resolution assets in lists.
- Animations on the UI thread (Reanimated / `useNativeDriver: true`); never animate layout
  with `setState` per frame.
- Memoise (`memo`, `useMemo`, `useCallback`) where a measurement or an obvious hot path
  (list rows, context values) justifies it — not by default. If the project uses the React
  Compiler, do not add manual memoisation.
- No heavy synchronous work on the JS thread during navigation transitions; defer with
  `InteractionManager` / `startTransition` when the project does.
- Avoid unnecessary re-renders from wide context values; split contexts or select from the
  store.
- New native dependency → confirm it supports the project's architecture (New Architecture /
  Expo Go vs dev client) and note whether a new dev-client build is needed.

## UX states, offline & error handling

- Every data-driven screen handles **loading**, **empty**, **error**, **offline** (when the
  project supports it) and **success**. Empty states say what to do next; errors are
  human-readable with a retry; pull-to-refresh where lists are refreshable.
- Forms: disable submit while pending, keep input on failure, field-level errors next to
  the field, success feedback (toast/haptic per project).
- Permissions: request lazily at the moment of use, after explaining why; handle *denied*,
  *blocked/never-ask-again* (link to settings) and *unavailable* (simulator) paths.
- Optimistic updates only where the project already does them and rollback is implemented.
- Unexpected errors reach the project's crash/error reporting (Sentry etc.) through the
  screen/root error boundary, not `console.error` alone. Never swallow a promise rejection.
- Background/foreground transitions (`AppState`) refresh stale data where the spec needs it.

## Security

- No secrets, API keys or internal URLs in the JS bundle or in `app.config` `extra` —
  anything there is public. Tokens live in SecureStore/Keychain.
- Auth state and tokens are attached by the API client, not by individual screens.
- Deep links / universal links validate their params before navigating or mutating.
- Sanitize anything rendered from user content in WebViews; restrict WebView origins.
- New permissions come with a usage-description string (iOS) / manifest entry (Android)
  and a one-line rationale in the docs; remove permissions the feature does not need.

## Testing bar

- Component/screen: React Native Testing Library + `userEvent`, rendered inside the
  project's test wrapper; assert what the user sees. Cover happy path, each
  validation/error branch, empty state, offline path when supported, and the
  permission-denied path for anything native.
- Hooks: `renderHook` with the same wrapper; test the contract, not the internals.
- Services/stores/utils: plain function tests; network mocked by MSW handlers in the test
  setup; native modules mocked at the wrapper boundary.
- Navigation: assert the destination screen (or `router.push` call via the project's mock)
  and the params, not internal navigator state.
- E2E (Maestro/Detox): only the critical journey named in the spec; `testID`s per project
  convention; wait on elements, never sleep.
- No tests that pass regardless of the implementation (assert outcomes, not calls, unless
  the call *is* the outcome).

## Documentation

- JSDoc on exported screens, components, hooks and services: what it does, non-obvious
  props/params, examples when usage is not self-evident. Do not document the obvious.
- New route/screen → the project's route/deep-link inventory; new env/config value →
  `.env.example` / `app.config.ts` docs; new permission → rationale + store-listing note.
- New native dependency or config plugin → note whether a new dev client / native build
  is required and how to produce it.
- If there is a Storybook, a story per visual state of a new shared component.
- Decisions that deviate from the spec or from the obvious approach go in the spec's
  **Agent summary → Key decisions**, with the reason, and manual QA steps per platform in
  **Manual QA**.

## Before you call the spec done

- [ ] `typecheck`, `lint`, full `test` green; project's bundle/native gate green if any
      (and `test:e2e` when required)
- [ ] Loading / empty / error / (offline) / success states present and tested
- [ ] Every tappable element has a role and an accessible name; targets ≥ 44 pt
- [ ] Safe areas and keyboard handled on the new screen, both platforms considered
- [ ] Lists virtualised with stable keys; images sized and cached
- [ ] No secrets in the bundle; new permissions justified and documented
- [ ] Docs, stories, `.env.example` / `app.config.ts` updated; spec and order file recorded
      (incl. Manual QA)
