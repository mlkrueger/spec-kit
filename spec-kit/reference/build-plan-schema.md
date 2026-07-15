# Build Plan Schema (the neutral contract)

> The **build plan** is the tracker-neutral hand-off between the `build-plan-architect` agent
> (producer) and an execution agent + tracker publishers (consumers). The architect emits
> `build-plan.yaml` beside `BUILD_PLAN.md`. It is a sibling to `ticket-plan` (*what to test*) and
> `acceptance-plan` (*what journeys must pass*); this one is **what to build, test-first**.
>
> **The plan holds intent only** — no tracker-specific fields. Each ticket is an *implementation* unit
> that carries its own unit/integration tests inline as TDD definition-of-done.

The machine-validatable definition is `build-plan.schema.json` (JSON Schema draft 2020-12). This
document is the prose companion. When the two disagree, the JSON Schema wins for field shape; this doc
wins for *meaning and process*.

## Object model

```
plan
├── version            # const 1
├── spec               # companion doc filename (BUILD_PLAN.md)
├── constraints        # constraint-envelope filename (constraints.yaml) — source for constraintRefs
├── title
├── milestone          # one top-level grouping (per-epic when fanning out; merged on emit)
└── tickets[]          # flat list — hierarchy via parent/blockedBy, not nesting
    └── ticket
        ├── key                 # REQUIRED. stable, kebab-case, UNIQUE
        ├── title               # REQUIRED
        ├── layer               # REQUIRED. feature|interface|migration|infra|refactor|harness|ci|docs
        ├── description         # REQUIRED. markdown body
        ├── interface?          # the contract to implement (signatures / request-response shapes)
        ├── tddCases            # REQUIRED for feature/interface/migration. unit/integration cases to write FIRST
        ├── modulesInScope      # REQUIRED. REAL verified paths
        ├── constraintRefs?     # keys from constraints.yaml this ticket must honor
        ├── tracesTo?           # PR-* product-requirement IDs (traceability spine)
        ├── tier?               # advisory: simple|standard|complex (model-routing hint)
        ├── phase?              # advisory: derived execution slice (capped leveling of blockedBy)
        ├── acceptanceCriteria  # REQUIRED. checkable DoD incl. the red->green->refactor loop
        ├── stack?, priority?, estimate?, labels?
        └── parent?, blockedBy?
```

## What's new vs. `ticket-plan`

`build-plan` shares `ticket-plan`'s conventions (kebab `key`, flat list + relationships, the four
non-schema constraints) but adds implementation-oriented fields:

- **`layer`** is implementation-oriented (`feature`/`interface`/`migration`/…), not test-pyramid.
- **`interface`** — the seam to implement, lifted from the technical spec's boundaries.
- **`tddCases`** — the unit/integration cases to write **before** the code (the red tests). The
  inline-TDD core: the failing test *is* the spec, in the same ticket. Required on the code-bearing
  layers (`feature`, `interface`, `migration`).
- **`constraintRefs`** — keys from `constraints.yaml` the ticket must honor.
- **`tracesTo`** — `PR-*` IDs the ticket helps satisfy.
- **`tier`** — an advisory model-routing hint (see below).

## Why tests live inline (not a separate plan)

TDD only works when the test and the code are authored together against the same seam. Splitting them
into separate tickets defeats red→green→refactor. So unit/integration tests are an **acceptance
criterion of each build ticket**, captured in `tddCases` + the inner-loop `acceptanceCriteria` — not a
standalone test plan. (E2E is the exception: it comes from the product spec, is authored up-front, and
lives in the separate `acceptance-plan`.) See `unit-integration-standards.md` and
`acceptance-standards.md`.

## The `tier` hint

`tier` lets the plan advise an orchestrator how to route execution **without committing to a model**:

- `simple` — mechanical, well-bounded (a CRUD path matching an interface, a pure-function module with
  clear cases). A cheap model can drive red→green→refactor unattended, because the inline `tddCases`
  fully specify the work.
- `standard` — normal feature work.
- `complex` — concurrency, security-sensitive, or cross-cutting → reserve a stronger model and/or human
  review.

The plan stays declarative; **what each tier maps to** (which model, subscription, or harness) lives in
per-team config, not in the plan — the same neutral-plan-plus-config split the publishers use. If you'd
rather keep the plan fully model-agnostic, omit `tier` and let the orchestrator infer from
`estimate`/`layer`.

## The `phase` field

`phase` slices *execution* (never the milestone — that stays one semantic unit) into dependency-honest
chunks an orchestrator can run with bounded context:

- **Derived, never authored**: a topological leveling of the `blockedBy` graph — phase 1 has no
  blockers; every ticket's phase exceeds each blocker's phase — greedily split so no phase exceeds the
  cap (`phaseCap`, default 10, per-team config). A pure function of the graph + cap.
- **All-or-none, contiguous from 1**: partial phasing is meaningless; `validate-build-plan` hard-fails
  mixed coverage, cap violations, numbering gaps, and any phase that contradicts a `blockedBy` edge
  (drift after editing dependencies without re-deriving).
- **Advisory and milestone-scoped**: dependencies stay authoritative; `phase` is read as
  *(milestone, phase)*, never globally. Publishers emit it as a bare `phase:K` label. The companion
  dev-orchestrator uses phases as natural respawn boundaries (layer K+1 depends only on ≤K); when the
  field is absent it falls back to its own count-based respawn — phases are guidance, not a
  correctness dependency.

## The done-gate ticket (always emitted)

Every build plan emits one `ci`-layer ticket wiring the **feature done-gate**: the inner suite (this
plan's unit/integration tests) **and** the outer `acceptance-plan` E2E job must both be green before
merge. This is where the two TDD loops — inner (build plan) and outer (acceptance plan) — are asserted
together.

## Constraints the validator checks beyond JSON Schema

`bin/validate-build-plan` runs the JSON Schema plus:

1. **`key` uniqueness** — every `ticket.key` unique; milestone key distinct.
2. **Referential integrity** — every `parent`/`blockedBy` references a real ticket key.
3. **Acyclicity** — the `parent`+`blockedBy` graph has no cycles.
4. **Real paths** — `modulesInScope` has no empty/placeholder entries.
5. **`constraintRefs` exist** — every referenced key exists in the constraint envelope
   (`--constraints`, the plan's `constraints:` field, or a sibling `constraints.yaml`).
6. **`tracesTo` exist** — every `PR-*` exists in the product spec (`--product-spec` or a sibling
   `PRODUCT_SPEC.md`).

Checks 5–6 make the traceability spine **enforced**, not aspirational. If a plan uses `constraintRefs`
or `tracesTo` but the corresponding source can't be found, that's a usage error (exit 2) — references
are validated against their source, never in a vacuum.

## Validation

```sh
${CLAUDE_PLUGIN_ROOT}/bin/validate-build-plan build-plan.yaml \
  --constraints constraints.yaml --product-spec PRODUCT_SPEC.md
```

A publisher should refuse to run on a plan that fails validation.

## Example

See `examples/build-plan.example.yaml` for a complete, valid plan: foundational `interface` and
`harness` tickets blocking the `feature` tickets that consume them, inline `tddCases` on every
code-bearing ticket, `constraintRefs` into `constraints.example.yaml`, `tracesTo` into the product
spec, all three `tier`s, and the always-emitted `ci` done-gate.
