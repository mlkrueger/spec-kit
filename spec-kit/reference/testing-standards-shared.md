# Testing Standards — Shared (Reference)

> The cross-layer testing discipline, shared by **both** downstream agents: `build-plan-architect`
> (which references it inline when writing each ticket's unit/integration `tddCases`) and
> `acceptance-spec-architect` (which references it when writing E2E tickets). It holds only what is
> true at *every* layer — ticket specificity, determinism, and the "a spec precedes the suite"
> discipline. Layer-specific judgment lives in `unit-integration-standards.md` and
> `acceptance-standards.md`. When a project's own conventions conflict with these, the project wins,
> but call out the divergence.

## Purpose of a test (applies at every layer)

A test serves two jobs: **a CI quality gate** (block a regression before merge) and **living
documentation** of intended behavior. Every test you specify must be justifiable against one of those.
If a test neither gates a real failure mode nor documents a real contract, cut it — it is noise.

## Determinism (non-negotiable, every layer)

Every suite must be **deterministic**: same inputs → same result, every run, on every machine. No live
external accounts, no real network, no secrets in CI, no wall-clock or timezone dependence. A flaky
test is worse than none — specify that flakes are quarantined or fixed immediately, never tolerated.
(The *mechanics* of achieving determinism differ by layer — clock injection and seam-mocking for
unit/integration, a mocked backend for E2E — and live in the two layer-specific docs.)

## A spec precedes the suite

Each test area gets an **enumerated case list — input → expected, including edges and the relevant
failure modes — before implementation.** This is what lets tests be written first (red) and what makes
a ticket implementable without re-deriving intent. For the inner loop the case list is the ticket's
`tddCases`; for the outer loop it is the E2E ticket's cases. Either way, no test is vague: each states
a concrete expected outcome.

## Naming signals the layer / environment

Use a file-naming scheme that routes each test to the correct runner/environment (e.g. a distinct
infix for DOM/component tests vs. node tests, or for E2E specs vs. unit specs), and document the rule
so a misnamed test doesn't silently run in the wrong environment. The exact infixes are a per-stack
profile detail; the *requirement* that naming be unambiguous is universal.

## Ticket structure (required in every ticket, every plan)

Every ticket a downstream agent emits — a build ticket, an E2E ticket, a refactor, a harness, a CI
gate — must be implementable **without re-exploring the codebase or re-reading the upstream spec**.
Beyond its case list / `tddCases`, each ticket MUST include:

- **Modules in scope** — the explicit, **real, verified** file paths an implementer will touch or
  read: the system-under-test module(s), the test file to create (with the correct naming infix for
  its runner/environment), and any shared helpers, fixtures, types, or config the test depends on.
  Never "the relevant files." If scope can only be known after reading code, make *recording those
  paths* the first acceptance item, and name the entry-point file to start from.
- **Acceptance criteria** — a checkable Definition of Done as a checklist, not a restatement of the
  cases. At minimum: every enumerated case implemented and passing; the file runs green in the correct
  test project/environment; no skipped or focused tests left behind; and all blocked-by prerequisites
  resolved. Add ticket-specific criteria as needed.

## Tracker-neutral plans, one validator family

Tickets are emitted as a **tracker-neutral plan** (YAML) holding **intent only** — no tracker-specific
fields (team, project key, issue type). Each plan kind (`build-plan`, `acceptance-plan`, and the
legacy `ticket-plan`) has its own schema and `bin/validate-*` tool, but all share the same conventions:

- A stable, unique, **kebab-case `key`** per ticket — load-bearing identity, stamped onto the created
  issue for idempotent re-publishing, and referenced by `parent` / `blockedBy`.
- A **flat ticket list**; hierarchy and ordering are expressed by `parent` ("belongs under") and
  `blockedBy` ("must come after") relationships, by `key` — never by nesting. Adapters translate those
  into each tracker's native structure.
- Every validator checks, beyond its JSON Schema: **key uniqueness**, **referential integrity** of
  `parent`/`blockedBy`, **acyclicity**, and **real (non-placeholder) paths** in `modulesInScope`.

## Track the work as discrete tickets

One ticket per coherent area/seam, with prerequisite refactors and shared harness/fixture setup
called out as explicit `blockedBy` blockers. A refactor that must precede a test, or a harness a suite
depends on, is its own ticket that blocks the dependent work — never an unstated assumption inside
another ticket.
