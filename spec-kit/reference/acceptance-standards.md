# Acceptance / E2E Standards (Reference)

> The outer-loop testing brain, referenced by `acceptance-spec-architect` when it turns a product
> spec's acceptance criteria into executable end-to-end tests. Covers **only** what is specific to the
> E2E layer — the considerations unit/integration tests don't have. Read alongside
> `testing-standards-shared.md` (ticket specificity, determinism, "a spec precedes the suite"), which
> holds the cross-layer discipline this document does not repeat. Lower layers live in
> `unit-integration-standards.md`. When a project's conventions conflict, the project wins; note the
> divergence.

## What the E2E layer is — and why it earns its own plan

End-to-end / acceptance tests exercise a handful of critical user journeys through the **running
system**: a real browser for web UIs, CLI invocation for command-line tools, or black-box API calls
for services. They are the **executable form of the product spec's acceptance criteria** — the outer
red→green loop for the whole feature.

They are a *separate, up-front plan* (`acceptance-plan.yaml`), distinct from the inline unit/integration
tests in the build plan, for four reasons:

1. **Different source of truth.** They derive from the **product spec** (what the user observes), not
   the technical spec (internal seams).
2. **Authored before any code exists.** The acceptance criteria are known at spec time, so these tests
   can be written first and act as the target the build plan turns green.
3. **They run as their own CI job**, separate from the fast unit/integration suite.
4. **They survive refactors.** Because they reference no internal module, they stay valid as the
   implementation changes underneath them.

## Test the journey, not the click

Each E2E test is a real **user-observable path**, written from the product spec's Given/When/Then
acceptance criteria — never a script of isolated UI clicks, and never a reference to an internal
module, function, or route handler. Assert what the user can observe (a confirmation appears, an email
is queued, the order is retrievable), not how the system achieved it. If a test names an internal
seam, it belongs in the integration layer, not here.

Trace every E2E test back to the `PR-*` requirement(s) whose acceptance criteria it exercises (the
ticket's `tracesTo`). An acceptance criterion in the product spec with no E2E test covering it is a
gap to flag.

## The mocked-backend harness

E2E here means **a built app against a mocked backend**, not a live production stack — that is what
keeps the layer deterministic (per the shared determinism rule):

- **Auth** — bypass via a seeded session or test-only login, never real credentials.
- **External HTTP** — intercept and stub third-party calls (payment processors, mail providers, feeds)
  at the network boundary with fixed responses, including their failure responses.
- **Datastore** — a fake or seeded store reset to a known fixture before each journey.
- **Time** — pin the clock where a journey depends on dates.

Standing up this harness (the built-app + mocked-backend rig, the seed fixtures, the auth bypass) is
its own `harness` ticket that **blocks** the journey tickets depending on it.

## Reserve it — E2E is the expensive, brittle layer

E2E is the most expensive to run and the most brittle to maintain. Keep the set **small and
high-value**: the critical journeys a failure of which would mean the feature is broken for users
(the happy path of each major requirement, plus the one or two highest-stakes error journeys — a
declined payment, a fail-closed authz). Push everything else down to unit/integration per "pick the
lowest layer that proves it." A sprawling E2E suite is a liability, not coverage.

## CI: a separate job, artifacts on failure

E2E runs as its **own CI job** against a built app + mocked backend — not in the fast
lint/type-check/unit/integration gate that must be green to merge. On failure, upload trace / video /
screenshots as artifacts for diagnosis. The whole-feature "done" gate requires **both** this
acceptance job and the build plan's inner suite to be green.

## Acceptance-ticket shape

Each E2E ticket follows the shared ticket structure (real `modulesInScope` paths — the spec file with
the correct E2E naming infix, the harness, the fixtures — and a checkable `acceptanceCriteria`
checklist), plus:

- **`cases`** — the journeys as Given/When/Then, lifted from the product spec's acceptance criteria.
- **`tracesTo`** — the `PR-*` requirement IDs the journey exercises.
- **`blockedBy`** — the harness ticket, and any seed/fixture ticket it depends on.
