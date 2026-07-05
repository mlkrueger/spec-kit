# Test Audit Standards (Reference)

> Reference document for the **test-audit-architect** agent: how to evaluate an **existing** test
> suite against the spec-kit testing standards, layer by layer. The forward-looking docs
> (`testing-standards-shared.md`, `unit-integration-standards.md`, `acceptance-standards.md`) define
> what *should* be built; this document defines how to judge what *was* built — the evidence to
> collect, the dimensions to score, and the shape of the findings. When a project's own conventions
> conflict with these, the project wins, but call out the divergence.

## The one discipline everything else serves

**Every finding is grounded in evidence, and every gap is stated as a missing observation.** A
finding cites the file/test it comes from and the standard it violates; a gap names the behavior,
seam, or failure mode that no test would catch if it broke. "Coverage is weak" is not a finding;
"`src/billing/proration.ts` has no test for the mid-cycle downgrade path — a regression there ships
silently (violates: failure-mode catalog, *business-rule limits*)" is. If you can't cite it or name
what breaks unobserved, don't write it.

## Scope — audit by layer

An audit is scoped to **unit**, **integration**, **e2e**, or **all** (default). Each layer is judged
against its own standards doc; the shared disciplines apply everywhere:

| Scope | Judged against |
|---|---|
| unit, integration (and component) | `unit-integration-standards.md` + `testing-standards-shared.md` |
| e2e | `acceptance-standards.md` + `testing-standards-shared.md` |
| all | the above, **plus the cross-layer dimensions** (pyramid shape, layer routing) that only make sense whole |

When scoped to one layer, still *inventory* the others briefly — you can't judge whether logic is
tested at the lowest layer that proves it without knowing what the other layers hold — but score and
report only the requested layer.

## Evidence to collect (before judging anything)

1. **Inventory, grounded.** Discover the real suites: runners and configs (from manifests/lockfiles →
   `profiles/<stack>.md`), test commands, file counts per layer, naming infixes in actual use, shared
   harnesses/fixtures, CI jobs that run them. Every claim from real files (brownfield principle 5) —
   read configs, don't infer them.
2. **Run the suites** (when the environment allows). Record pass/fail, runtime, and — cheap and
   high-signal — a **second run** to catch order-dependence and flakiness. Collect coverage on the
   logic core if the runner supports it without heroics. If suites can't run, say so; an audit of
   tests that don't run starts there.
3. **Map tests to behavior.** For the scoped layer(s), list the significant modules/seams (or, for
   e2e, the user journeys — from `features/*/PRODUCT_SPEC.md` acceptance criteria when specs exist,
   from routes/flows when not) and which tests exercise each. The unmapped remainder is the gap list.

## Scoring dimensions

Score each as **sound / weak / absent**, with the evidence that earned the score.

**Every layer** (from `testing-standards-shared.md`):
- **Determinism** — real network/clock/timezone/external-account dependence; order-dependence;
  observed flakes; a flake policy (quarantine-or-fix) or its absence.
- **Naming & routing** — does the naming scheme route each test to the right runner/environment, and
  is it followed? Misrouted tests run in the wrong world silently.
- **Signal** — tests that neither gate a real failure mode nor document a contract (asserting
  mock-echoes, snapshot noise, tautologies) are inventory, not coverage. Name them.

**Unit / integration / component** (from `unit-integration-standards.md`):
- **Layer placement** — is branchy logic tested at the lowest layer that proves it, or trapped behind
  integration/E2E harnesses? Logic reachable only through a high layer is a refactor finding.
- **Mocking discipline** — mocked at one well-defined seam vs. deep mock-patching internals;
  hand-rolled fakes where argument assertion matters; shared harness vs. copy-pasted setup.
- **Clock & state** — injected time vs. real clock; DST/boundary cases; shared mutable state between
  tests; teardown hygiene.
- **Contract vs. implementation** — assertions on observable outputs and boundary-crossing arguments
  vs. private internals a refactor will churn.
- **Error branches** — the dependency-errored / invalid-input path tested wherever a happy path is;
  **fail-closed** proven for authz-like decisions.
- **Failure-mode catalog** — audit the covered modules against each catalog category (async races,
  optimistic UI, time/DST, fail-closed security, environment gating, limits & caps, data
  propagation, input edges); list the categories with no test anywhere they apply.
- **Coverage where it matters** — thresholds on the logic core vs. a global number dominated by
  trivial code. Report the core modules' real numbers, not the vanity aggregate.

**E2E** (from `acceptance-standards.md`):
- **Journey vs. click** — user-observable Given/When/Then paths vs. click-scripts or tests naming
  internal modules/routes.
- **Harness determinism** — auth bypass, intercepted external HTTP (incl. failure responses), seeded
  store reset per journey, pinned clock — vs. live dependencies (an E2E suite hitting real services
  is a determinism finding, not a harness).
- **Reserved set** — small and high-value vs. sprawl; anything here that a lower layer could prove
  belongs down there (and its runtime cost says so).
- **CI separation** — own job, separate from the fast merge gate; failure artifacts
  (trace/video/screenshots) uploaded.
- **Traceability** — where product specs exist: acceptance criteria with no journey, journeys tracing
  to no requirement.

**Cross-layer** (scope=all only):
- **Pyramid shape** — the distribution across layers vs. where the logic lives: inverted pyramids
  (heavy E2E, hollow unit), missing middles (units + E2E but no wiring tests), duplicated coverage
  (the same behavior asserted at three layers).

## Findings, not vibes

Each finding: **evidence** (file/test, run output) → **standard violated** (doc + section) →
**consequence** (what ships broken / what regression goes uncaught) → **remediation** (the concrete
change) → **severity**: `high` (a real failure mode ships undetected: fail-open authz untested, flaky
merge gate, suites that don't run), `medium` (a standards violation that will bite: real clock,
mock-echo tests, missing error branches), `low` (hygiene: naming drift, redundant coverage).

## From audit to backfill plan (the hand-off)

The audit's remediations become an **ordinary `build-plan.yaml`** — same schema, same
`validate-build-plan`, same publishers — so the backfill is executable and trackable like any feature
work. Conventions for a backfill plan:

- **Ticket layers**: test-writing tickets are `feature`-layer (their `tddCases` are the enumerated
  cases to add — the case list *is* the work); testability refactors (extract pure functions, inject
  the clock, expose a seam) are `refactor` tickets that **block** the tests needing them; missing
  harnesses are `harness` tickets; CI wiring (coverage floor, E2E job split) is `ci`.
- **Characterization discipline** (the backfill twist on red→green): a test written for *existing*
  behavior should pass immediately. One that fails has found either a wrong assumption — fix the
  test — or a **real bug: file it as its own finding/ticket, never "fix" the test to match broken
  behavior, and never quietly change the behavior inside a backfill ticket.** The acceptance criteria
  on backfill tickets encode this instead of the standard red→green clause.
- **Traceability**: where product specs exist, trace backfill tickets to the `PR-*` whose behavior
  they pin; where none exist, omit `tracesTo` (the validator then requires no product spec) and note
  the plan is audit-derived — the audit document is its source of truth.
- **Priority mirrors severity**; keys are prefixed `test-audit-` (or `<feature>-test-audit-` in a
  feature-scoped repo) to stay collision-free.
