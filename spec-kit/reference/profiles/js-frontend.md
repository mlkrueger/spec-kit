# Profile: JS/TS Frontend (Vitest + jsdom, Playwright)

Test stack for TypeScript/JavaScript **browser** apps — the component-bearing frontend (React, Svelte,
Vue, SvelteKit, etc.). The DOM layer and the E2E layer. For a Node backend in the same repo, use the
`js-node` profile alongside this one (the repo is polyglot across the two keys).

## Runner & commands

- **Unit/component:** `vitest` (DOM environment). **E2E:** `@playwright/test`.
- **Whole unit suite:** `vitest run` (watch: `vitest`)
- **One file:** `vitest run src/foo.component.test.ts`
- **One test:** `vitest run -t "test name"`
- **Coverage:** `vitest run --coverage` (v8 provider)
- **E2E suite:** `playwright test`; **one spec:** `playwright test e2e/foo.e2e.ts`

## Layers (which the build/acceptance agents target)

- **Unit** (build plan, inline `tddCases`) — pure functions, stores/state logic, formatters,
  validation. Fastest, most numerous; no DOM needed.
- **Component** (build plan, inline `tddCases`) — render components in a headless DOM
  (`@testing-library/*` + `jsdom`/`happy-dom`), asserting observable output from props/state. Keep to
  what a user can see; never assert internals.
- **End-to-end** (acceptance plan) — a handful of critical journeys in a real browser against a
  **built app + mocked backend** (seeded session / auth bypass, intercepted HTTP via Playwright
  `route`).

## File naming & location

- Co-locate unit tests as `*.test.ts` next to source.
- **Route DOM/component tests to the DOM environment explicitly** — a distinct infix
  (`*.component.test.ts`) or a `// @vitest-environment jsdom` pragma — so a misnamed test doesn't run in
  the node environment and silently behave differently. This infix is what a build ticket's
  `modulesInScope` component-test path must use.
- E2E specs in `e2e/`, named `*.e2e.ts` (or `*.spec.ts` per the repo); shared fixtures (auth, network,
  seed) in `e2e/fixtures/` — the `modulesInScope` an acceptance ticket references.

## Mocking seam & determinism

- Inject the dependency surface (a `makeContext()` / fake-client factory) rather than reaching for
  module mocks where possible; use MSW for HTTP at the network boundary.
- **Inject the clock** — pass `now`; for timers use `vi.useFakeTimers()`. Lock `TZ=UTC` in CI and add
  DST cases.
- Vitest does not auto-reset mocks unless configured — `vi.restoreAllMocks()` in teardown or set
  `restoreMocks: true`.

## Gotchas to encode as regression tests

- **Framework reactivity** — Svelte 5 runes (`$state`/`$derived`/`$effect`) update asynchronously; await
  the tick / use `flushSync` before asserting. React effect timing and stale closures. (Per
  cross-project style, new Svelte code uses runes, not Svelte 4 patterns.)
- **Lifecycle / async races** — submit-then-navigate; a component unmounts before an `await` resolves
  and silently cancels the request; double-submit.
- **Optimistic UI vs. revalidation** — after a local optimistic mutation, the server refresh must not
  flicker, revert, or double-apply.
- **SSE / streaming** — partial chunks, reconnection, abort on unmount.
- **Env gating** — dev-only conveniences (auth bypass, seed/debug routes) must be provably absent from a
  production build.
- **Date handling** — local-time display over UTC storage; `Date` parsing differences; day/week
  boundary math.
- **Serialization** — `undefined` vs `null` across the JSON boundary; `Date` → string round-trips.
