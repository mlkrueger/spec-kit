# Build & Test Execution Agents — Design

> Extends `SPEC_AGENTS_DESIGN.md`. Where that doc designed the *upstream* spec agents
> (product → technical), this designs the *downstream* decomposition and TDD-execution layer:
> turning specs into real, buildable, test-driven tickets an execution agent (potentially a
> Haiku-class model) can walk through to build the system.
>
> Mirrors the established `test-spec` conventions: agent body in `agents/*.md`, judgment in
> `reference/*-standards.md`, a tracker-neutral plan validated by a `bin/validate-*` tool,
> project-scoped agent memory, produce-then-hand-off discipline.

---

## 1. The core insight: "tests" and "tickets" are each two jobs

The single `test-spec-architect` conflates work that comes from two different sources and runs at
two different times:

| Test kind | Derived from | When written | Nature |
|---|---|---|---|
| **E2E / acceptance** | **Product spec** | Up-front, before implementation | Black-box, user-observable journeys, stack-light. The executable form of acceptance criteria. |
| **Unit / integration** | **Technical spec** + module decomposition | Interleaved, per-module | White-box, references real internal seams (injected clock, repo interface). |

And the **missing downstream artifact** is the *implementation plan* — the actual buildable tickets
an execution agent walks through.

So the spec phase needs **two new agents**, with the existing test-spec agent **narrowed to a
standards reference**:

```
PRODUCT_SPEC.md ─────────────▶ acceptance-spec-architect ──▶ ACCEPTANCE_SPEC.md
   │                                                          + acceptance-plan.yaml  (E2E, up-front)
   │
   ▼
TECHNICAL_SPEC.md ──▶ build-plan-architect ──▶ BUILD_PLAN.md + build-plan.yaml
   (+ constraints.yaml)        │                 (impl tickets, each with INLINE unit/integration
   (+ PRODUCT_SPEC.md)         │                  tests as TDD definition-of-done)
                               │
                               └── applies ──▶ reference/testing-standards.md  (the old test-spec
                                                                                 agent's brain,
                                                                                 now pure reference)
```

---

## 2. Why unit/integration tests live INLINE in build tickets (the key decision)

TDD only works when the test and the code are authored **together against the same module seam**.
Splitting "write the test for module X" and "build module X" into separate tickets/agents creates a
coordination seam that defeats red→green→refactor, and forces the executor to context-switch
between two plans.

**Therefore:**

- **Unit & integration tests are an *acceptance criterion* of each build ticket**, not a separate
  plan. A build ticket isn't done until its tests were written first (red), the code makes them
  pass (green), and it's refactored clean. This is what lets a **Haiku-class executor** drive the
  loop — the failing test *is* the spec, in the same ticket.
- **E2E / acceptance tests are a separate, up-front plan.** They come from a different source of
  truth (the product spec), can be authored before any code exists, run as a separate CI job, and
  act as the **outer red→green** for the whole feature. They earn their own agent.

This is the "hybrid" model: inline unit/integration + standalone E2E + retained standards doc.

---

## 3. Agent A — `build-plan-architect` (NEW)

**Role.** A staff engineer / tech lead who decomposes an approved technical spec into a dependency-
ordered set of **implementation tickets** precise enough that an execution agent can build the
system ticket-by-ticket with TDD, without re-deriving the design.

**Inputs.** `TECHNICAL_SPEC.md` (primary), `constraints.yaml` (hard/soft bounds it must respect and
propagate), and `PRODUCT_SPEC.md` (for the `PR-*` requirement IDs each ticket traces to). Optionally
an existing codebase to extend.

**Output.** `BUILD_PLAN.md` (human-readable narrative + sequencing rationale) and `build-plan.yaml`
(the machine-readable, execution-ready plan).

**Core discipline.**
> **Every ticket is independently buildable and test-first.** A ticket states the interface to
> implement, the unit/integration tests to write *before* the code, the real files in scope, the
> constraints it must honor, and a checkable DoD. If an execution agent would have to re-read the
> tech spec to start, the ticket is underspecified.

### What makes a build ticket "execution-ready"

Each ticket carries, beyond title/description:

- **`interface`** — the contract to implement (function/class/endpoint signatures, request/response
  shapes), taken from the tech spec's boundaries. Not full code — the seam.
- **`tddCases`** — the unit/integration cases to write **first** (input → expected, incl. edges and
  the failure-mode catalog from `testing-standards.md`). These are the red tests.
- **`modulesInScope`** — real, verified paths: the module(s) to create/edit, the test file (correct
  naming infix), shared fixtures/harness.
- **`constraintRefs`** — the `constraints.yaml` keys this ticket must honor (e.g. `lang-python`,
  `db-postgres`), so the executor and a linter can check compliance.
- **`tracesTo`** — the `PR-*` product-requirement IDs this ticket helps satisfy (the traceability
  spine).
- **`acceptanceCriteria`** — checkable DoD: tests written first and passing, coverage floor on
  logic-core, lint/type-check green, constraints honored, no skipped tests.
- **`tier`** — *optional* model-tier hint (`simple` / `standard` / `complex`) so an orchestrator
  can route `simple` tickets to a cheap model (Haiku) and reserve a stronger model for `complex`
  ones. The plan stays otherwise model-agnostic; `tier` is advisory.
- **`blockedBy` / `parent`** — same relationship model as `ticket-plan`: a migration blocks the
  feature that needs it; a shared interface blocks its consumers.

### Decomposition method (in the agent body)

1. **Read the tech spec's component boundaries** → one (or a few) tickets per module/seam.
2. **Order by dependency** → foundational seams (interfaces, data model, migrations, shared
   harness) first; consumers `blockedBy` them. Produce a buildable topological order.
3. **Size for autonomy** → each ticket is a coherent unit of behavior small enough to build+test in
   one pass, large enough to be meaningful. Split anything that needs two unrelated seams.
4. **Push tests down** → reuse the `testing-standards.md` "lowest layer that proves it" rule:
   inline unit tests by default, integration only at real wiring seams.
5. **Propagate constraints** → attach `constraintRefs`; never silently violate a hard constraint.
6. **Tag traceability + tier**, then **validate**.

### `reference/build-standards.md` (its standards-as-data)

- Ticket-sizing heuristics (one seam, one coherent behavior; the "no re-reading the spec" test).
- The red→green→refactor DoD template every ticket must satisfy.
- Dependency-ordering rules (foundational seams first; what may run in parallel).
- A failure-mode catalog for *plans*: tickets with vague scope, hidden cross-ticket coupling,
  missing interface contracts, tests that can't be written before the code (→ flag a refactor
  ticket, same as test-spec does), constraints with no `constraintRefs`.

---

## 4. Agent B — `acceptance-spec-architect` (NEW; the E2E split-out)

**Role.** A QA/acceptance engineer who turns the **product spec** into executable end-to-end /
acceptance tests — the outer TDD loop for the whole feature, writable before any code exists.

**Input.** `PRODUCT_SPEC.md` (its acceptance criteria + user journeys are the source). Optionally the
tech spec, only to learn the e2e harness/runner (built app + mocked backend).

**Output.** `ACCEPTANCE_SPEC.md` + `acceptance-plan.yaml` (a tracker-neutral plan of E2E tickets,
same schema family as `ticket-plan`, `layer: e2e`/`harness`).

**Core discipline.**
> **Test the journey, not the click.** Each test is a real user-observable path from the product
> spec's acceptance criteria (Given/When/Then), against a built app with a mocked backend — no
> reference to internal modules.

**Why separate from build tickets:** different source of truth (product, not technical), authored
up-front, runs as its own CI job, and stays valid across implementation refactors. It is the
acceptance gate the build plan is trying to turn green.

This is also the cleanest home for what your current `test-spec-architect` does at the E2E layer —
so `acceptance-spec-architect` effectively **inherits the E2E half** of the existing agent.

---

## 5. What happens to the existing `test-spec-architect`

**Narrow it to the unit/integration standards reference.** Its deep knowledge —
`testing-standards.md`, the failure-mode catalog, determinism/mocking rules, the four-layer pyramid
— stays and becomes the **brain the `build-plan-architect` applies inline** when writing each
ticket's `tddCases`. Two viable end states:

- **(Recommended) Retire the standalone unit/integration *plan* output; keep the standards.** The
  agent's reference docs live on; `build-plan-architect` cites them. E2E moves to
  `acceptance-spec-architect`. No more separate unit/integration ticket plan — those tests are
  inline DoD.
- **(Lighter touch) Leave it intact** for teams that still want a standalone test plan, and let the
  build agent simply reference it. Accept some overlap.

Either way, nothing about the existing agent is *lost* — its judgment is reused.

---

## 6. The build-plan schema (new sibling to `ticket-plan`)

A separate `build-plan.schema.json`, sharing the `ticket-plan` conventions (kebab-case unique
`key`, flat list + `parent`/`blockedBy` relationships, idempotency stamping, the four non-schema
constraints) but with implementation-oriented fields. New/changed fields vs. `ticket-plan`:

```
ticket
├── key, title, description            # as today
├── layer            # feature | migration | infra | interface | refactor | harness | ci | docs
├── interface?       # NEW. the contract to implement (signatures / request-response shapes)
├── tddCases         # NEW. unit/integration cases to write FIRST (input -> expected, + edges)
├── modulesInScope   # real verified paths (as today)
├── constraintRefs?  # NEW. keys from constraints.yaml this ticket must honor
├── tracesTo?        # NEW. PR-* product-requirement IDs (traceability spine)
├── tier?            # NEW, optional. simple | standard | complex (advisory model-routing hint)
├── acceptanceCriteria  # checkable DoD incl. "tests written first & passing"
├── priority?, estimate?, labels?, stack?
└── parent?, blockedBy?
```

`bin/validate-build-plan`: JSON Schema + the four non-schema checks (key uniqueness, referential
integrity, acyclicity, real paths) **plus two new ones**: every `constraintRefs` key exists in
`constraints.yaml`, and every `tracesTo` ID exists in the product spec. This makes the traceability
spine *enforced*, not aspirational.

Keeping it a separate schema (rather than overloading `ticket-plan`) keeps each plan's intent clean:
`ticket-plan` = "what to test," `build-plan` = "what to build (test-first)," `acceptance-plan` =
"what journeys must pass." Three narrow contracts, one shared validator pattern.

---

## 7. The model-tier hint (`tier`) — your "Haiku for simple tickets" goal

`tier` lets the plan advise an orchestrator how to route execution without committing to a model:

- `simple` — mechanical, well-bounded (CRUD endpoint matching an interface, a pure-function module
  with clear cases) → a Haiku-class agent can do red→green→refactor unattended.
- `standard` — normal feature work.
- `complex` — tricky concurrency, security-sensitive, or cross-cutting → reserve a stronger model
  and/or human review.

The plan stays declarative; the orchestrator decides. Because each `simple` ticket ships its
`tddCases` inline, the failing test fully specifies the work — which is exactly what makes a cheap
model viable. (If you'd rather keep the plan 100% model-agnostic, drop `tier` and let the
orchestrator infer complexity from `estimate`/`layer`.)

---

## 8. End-to-end flow (all five agents)

```
brief ─▶ product-spec-architect ─▶ PRODUCT_SPEC.md ──┬─▶ acceptance-spec-architect ─▶ acceptance-plan.yaml
                                                      │                                 (E2E, up-front, outer red)
                                                      ▼
                         technical-spec-architect ─▶ TECHNICAL_SPEC.md + constraints.yaml
                                                      │   (reviewed/iterated by a principal/staff eng here)
                                                      ▼
                              build-plan-architect ─▶ build-plan.yaml
                                                      │   (impl tickets, inline unit/integration TDD,
                                                      │    constraintRefs, tracesTo, tier)
                                                      ▼
                          execution agent (Haiku-class for `simple` tiers) walks tickets:
                          for each ticket → write tddCases (red) → implement (green) → refactor
                                                      ▼
                              feature complete when build tickets green AND acceptance-plan green
```

Two TDD loops, cleanly separated: **inner** (per-ticket unit/integration, owned by the build plan)
and **outer** (whole-feature E2E, owned by the acceptance plan). The principal/staff-eng review you
described slots in naturally between the technical spec and the build plan.

---

## 9. Open questions worth deciding before building files

1. **One plugin or several?** This pushes the plugin to 5 agents. Strongly favors the single
   `spec-kit` plugin from `SPEC_AGENTS_DESIGN.md` so all plans share one validator family and the
   traceability spine. (Could also publish build/acceptance plans to Linear via clones of your
   existing `publish-linear` skill — the neutral-plan pattern already supports it.)
2. **Does the build plan emit *one* big plan or one-per-epic?** For large systems, consider the
   build agent fanning out one sub-plan per technical-spec component (parallel sub-agents), merged —
   the same future-fan-out seam your test-spec README anticipates.
3. **Inner+outer green gate.** Decide where "feature done" is asserted: a CI meta-gate that requires
   both the build-plan tickets and the acceptance-plan to be green.
4. **Constraint linting.** If you want hard constraints *enforced* on generated code/IaC (not just
   referenced), that's a small `lint-constraints` tool reading `constraintRefs` — worth scoping
   separately.
