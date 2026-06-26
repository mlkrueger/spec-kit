# Acceptance Plan Schema (the neutral contract)

> The **acceptance plan** is the tracker-neutral hand-off between the `acceptance-spec-architect` agent
> (producer) and an execution agent + tracker publishers (consumers). The architect emits
> `acceptance-plan.yaml` beside `ACCEPTANCE_SPEC.md`. It is a sibling to `build-plan` (*what to build,
> test-first*) and `ticket-plan` (*what to test*); this one is **what journeys must pass** — the outer
> TDD loop, authored up-front from the product spec.
>
> **The plan holds intent only** — no tracker-specific fields. Every E2E ticket is a user-observable
> journey, never a reference to an internal module.

The machine-validatable definition is `acceptance-plan.schema.json` (JSON Schema draft 2020-12). This
document is the prose companion. When the two disagree, the JSON Schema wins for field shape; this doc
wins for *meaning and process*.

## Object model

```
plan
├── version            # const 1
├── spec               # companion doc filename (ACCEPTANCE_SPEC.md)
├── title
├── milestone          # one top-level grouping
└── tickets[]          # flat list — ordering via parent/blockedBy, not nesting
    └── ticket
        ├── key                 # REQUIRED. stable, kebab-case, UNIQUE
        ├── title               # REQUIRED
        ├── layer               # REQUIRED. e2e | harness
        ├── description         # REQUIRED. markdown body
        ├── cases               # REQUIRED for e2e. journeys as Given/When/Then (user-observable)
        ├── tracesTo            # REQUIRED for e2e. PR-* IDs the journey exercises
        ├── modulesInScope      # REQUIRED. REAL verified paths (E2E spec, harness, fixtures)
        ├── acceptanceCriteria  # REQUIRED. checkable DoD
        ├── stack?, priority?, estimate?, labels?
        └── parent?, blockedBy?
```

## Two layers only

- **`e2e`** — a user-observable journey against a built app + mocked backend. MUST carry `cases`
  (Given/When/Then, lifted from the product spec's acceptance criteria) and `tracesTo` (the `PR-*` it
  exercises). Asserts only what the user observes — no internal module, function, or route reference.
- **`harness`** — the mocked-backend rig and seed fixtures the journeys depend on (built app, auth
  bypass, intercepted external HTTP, fake/seeded store). Each `e2e` ticket `blockedBy` it.

## Why this is a separate plan from the build plan

Different source of truth and lifecycle, so it earns its own plan and validator:

- Derived from the **product spec** (user-observable), not the technical spec (internal seams).
- **Authored up-front**, before any code exists — it is the target the build plan turns green.
- Runs as its **own CI job**, separate from the fast unit/integration gate.
- **Survives refactors**, because it references no internal module.

See `acceptance-standards.md` for the full discipline (journey-not-click, the mocked-backend harness,
reserve-it, the separate CI job).

## Constraints the validator checks beyond JSON Schema

`bin/validate-acceptance-plan` runs the JSON Schema plus:

1. **`key` uniqueness** — every `ticket.key` unique; milestone key distinct.
2. **Referential integrity** — every `parent`/`blockedBy` references a real ticket key.
3. **Acyclicity** — the `parent`+`blockedBy` graph has no cycles.
4. **Real paths** — `modulesInScope` has no empty/placeholder entries.
5. **`tracesTo` exist** — every `PR-*` exists in the product spec (`--product-spec` or a sibling
   `PRODUCT_SPEC.md`).

Because `e2e` tickets must carry `tracesTo`, a product spec is effectively required; a plan that
references requirements with no spec to check them against is a usage error (exit 2) — journeys are
validated against the product they prove, never in a vacuum.

## Validation

```sh
${CLAUDE_PLUGIN_ROOT}/bin/validate-acceptance-plan acceptance-plan.yaml --product-spec PRODUCT_SPEC.md
```

A publisher should refuse to run on a plan that fails validation.

## Example

See `examples/acceptance-plan.example.yaml`: one `harness` ticket blocking three `e2e` journeys (the
happy path, the highest-stakes declined-payment error journey, and the lookup with its no-enumeration
edge), each tracing to the guest-checkout `PR-*` IDs.
