# Profile: JS/TS Node backend (Vitest, node env)

Test stack for TypeScript/JavaScript **Node backends** — services, APIs, workers, CLIs. No DOM. For a
browser frontend in the same repo, use the `js-frontend` profile alongside this one.

## Runner & commands

- **Unit/integration:** `vitest` (node environment — the default; do not pull in jsdom).
- **Whole suite:** `vitest run` (watch: `vitest`)
- **One file:** `vitest run src/orders/store.test.ts`
- **One test:** `vitest run -t "test name"`
- **Coverage:** `vitest run --coverage` (v8 provider)
- **End-to-end:** black-box HTTP against the built service — `vitest` (or the repo's runner) driving
  real requests at a started app with a mocked backend; or `supertest` against the app instance. No
  browser.

## Layers (which the build/acceptance agents target)

- **Unit** (build plan, inline `tddCases`) — pure functions, domain logic, validation, serialization,
  date math. Fastest, most numerous.
- **Integration** (build plan, inline `tddCases`) — route handlers / service functions / repositories
  with the datastore and external HTTP mocked at one seam (MSW or an injected client). This is where
  most backend regressions are caught.
- **Component** — N/A (no DOM).
- **End-to-end** (acceptance plan) — black-box HTTP journeys against the built app with a mocked
  backend (seeded store, intercepted third-party HTTP, test-only auth bypass).

## File naming & location

- Co-locate tests as `*.test.ts` next to source, or under `tests/` mirroring the layout. Distinguish
  integration with an infix (`*.integration.test.ts`) where it helps CI target them — the infix a build
  ticket's `modulesInScope` test path must use.
- E2E specs under `e2e/` or `tests/e2e/`, named `*.e2e.ts`; shared harness (started app, seed, HTTP
  intercept) in `e2e/support/` — the `modulesInScope` an acceptance ticket references.

## Mocking seam & determinism

- Inject the dependency surface (clients, repositories, clock) via constructor/params; prefer
  hand-rolled fakes when asserting call *arguments* (the query built, the payload sent). Use MSW for
  outbound HTTP.
- **Inject the clock** — pass `now`; `vi.useFakeTimers()` for timers. Lock `TZ=UTC` in CI; add DST
  cases.
- Reset state between tests — `restoreMocks: true` or `vi.restoreAllMocks()` in teardown; reset the
  fake store. Tests must pass in any order.

## Gotchas to encode as regression tests

- **Async / promise races** — un-awaited promises silently passing; unhandled rejections; concurrent
  writes; cancellation/abort paths.
- **Fail-closed security** — when an authz/allowlist source errors, deny — never default-open. Test the
  error path explicitly.
- **Env gating** — dev-only seed/debug/auth-bypass routes must be provably absent from a production
  build.
- **Input validation edges** — malformed JSON, empty/whitespace, oversized/ragged payloads, duplicate
  keys, unicode/BOM/line-ending variance.
- **Serialization** — `undefined` vs `null` across the JSON boundary; `Date` → string round-trips;
  number precision (money in integer cents, not floats).
- **Transaction boundaries** — partial-failure/rollback paths in multi-step writes; record-iff-confirmed
  invariants.
