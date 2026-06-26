# Unit & Integration Standards (Reference)

> The lower-layer testing brain, referenced **inline** by `build-plan-architect` when it writes each
> build ticket's `tddCases`. Covers unit, integration, and component layers — everything below E2E.
> Read alongside `testing-standards-shared.md` (ticket specificity, determinism, "a spec precedes the
> suite"), which holds the cross-layer discipline this document does not repeat. E2E lives in
> `acceptance-standards.md`. When a project's conventions conflict, the project wins; note the
> divergence.

## Why these tests live inline in build tickets

TDD only works when the test and the code are authored **together against the same module seam**.
Splitting "write the test for module X" and "build module X" into separate tickets creates a
coordination seam that defeats red→green→refactor. So unit and integration tests are an **acceptance
criterion of each build ticket**, not a separate plan: the ticket isn't done until its `tddCases` were
written first (red), the code makes them pass (green), and it's refactored clean. The failing test
*is* the spec, in the same ticket — which is what lets a cheap executor drive the loop unattended.

## The lower layers

1. **Unit** — pure functions and isolated modules, no I/O. The branchy business logic (parsers,
   date/recurrence math, aggregation, validation, state machines) lives here and should approach
   exhaustive coverage. Highest value-per-cost.
2. **Integration** — wiring between units with **all external dependencies mocked** (DB, third-party
   APIs, auth providers, file/feed sources). Request handlers, data-resolution layers, auth guards,
   validation/redirect logic — exercised by injecting a fake dependency surface. This is where most
   *regressions* are caught.
3. **Component** — UI components in a headless DOM, asserting rendered output and behavior from
   props/state, not internals. Keep to what a user can observe. (Applies only to projects with a UI.)

**Pick the lowest layer that can prove the behavior.** When logic is trapped somewhere only an
integration or E2E harness can reach, recommend a refactor to expose it (extract pure functions,
export helpers, inject `now`/a clock and dependencies) *before* specifying the test. Flag the
extraction as a prerequisite `refactor` ticket that **blocks** the dependent test work — never write a
`tddCase` that can't be written before the code it specifies.

## Determinism & mocking (the mechanics)

The shared doc requires determinism; here is how to achieve it at these layers:

- **Mock every external dependency** at a single, well-defined seam. Prefer a small, readable,
  hand-rolled fake over an auto-mock when tests need to assert *arguments* (the query built, the
  payload sent), not just return values.
- **Inject time.** Never let logic under test call the real clock — pass `now`/a clock. Lock a fixed
  timezone in CI and add explicit DST / boundary cases.
- **No shared mutable state between tests.** Know whether the runner auto-clears mocks; if not, reset
  in teardown. Tests must pass in any order and in isolation.
- **Assert the contract, not the implementation** — observable outputs and the arguments crossing a
  boundary, not private internals a refactor will churn.
- **Cover the error branch.** Any code with a happy path and a "dependency errored / invalid input"
  path must test both. Masked or unhandled errors are a top production failure mode.

## One shared test harness

Provide a single shared harness for the common dependency surface (a `makeContext()` / fake-client
factory) so integration tests stay short and readable. Setting it up is its own `harness` ticket that
blocks the tests depending on it.

## Coverage

Enforce **coverage thresholds on the logic core** (server/business modules), not chased globally — a
high global number dominated by trivial code hides gaps where they matter. Recommend a real floor
(e.g. 80% lines/branches) on the modules carrying business logic, and make meeting it an acceptance
criterion on those tickets.

## Encode known failure modes as regression tests (the catalog)

Whenever a bug, a subtle invariant, or a "you have to know this" gotcha surfaces, pin it with a test at
the lowest layer that can reach it. These categories recur across almost any app — audit each build
ticket's behavior against them and add `tddCases` that fail if the invariant breaks:

- **Framework lifecycle / async races** — submit-then-navigate, unmount-during-pending-request,
  double-submit (a form that unmounts before an `await` resolves and silently cancels the request).
- **Optimistic UI vs. revalidation** — after a local optimistic mutation, the server refresh must not
  flicker, revert, or double-apply.
- **Time, timezone & DST** — anything mixing local-time display with UTC storage; day/week boundaries;
  recurrence and overrides.
- **Fail-closed security** — when an authz/allowlist source errors, the system must deny, never
  default-open. Test the error path explicitly.
- **Environment gating** — dev-only conveniences (auth bypass, seed routes, debug endpoints) must be
  provably absent from a production build.
- **Business-rule limits & caps** — max counts, min-one constraints, quotas: test at, below, and above
  the boundary.
- **Data propagation / carry-forward** — operations that copy or migrate state (carry-forward, clone,
  import, archive-with-fallthrough) must preserve the right fields, mark the source correctly, respect
  caps, and handle the bulk/partial-failure path.
- **Input validation edges** — malformed dates, empty/whitespace, ragged/oversized input, duplicate
  keys, unicode/BOM/line-ending variance.

## The red→green→refactor DoD (every build ticket)

A build ticket's acceptance criteria must encode the inner loop explicitly:

- [ ] `tddCases` written **first** and failing (red) before implementation.
- [ ] Implementation makes them pass (green); every enumerated case passing.
- [ ] Refactored clean with tests still green.
- [ ] Logic-core coverage floor met; lint and type-check green.
- [ ] No skipped or focused tests; constraints in `constraintRefs` honored.
