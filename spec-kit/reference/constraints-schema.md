# Constraints Schema (the machine-readable envelope)

> The **constraint envelope** is the machine-readable hand-off between the `technical-spec-architect`
> agent (producer) and every downstream consumer — the `build-plan-architect`, the
> `acceptance-spec-architect`, and any coding/IaC agent. The architect emits `constraints.yaml`
> beside `TECHNICAL_SPEC.md`. Downstream agents read the **hard** constraints as non-negotiable
> givens and the **soft** constraints as overridable defaults.
>
> **The file holds the bounding decisions only.** The design narrative and rationale prose live in
> the spec; this file is the structured, lintable form of its Constraints section.

The machine-validatable definition is `constraints.schema.json` (JSON Schema draft 2020-12). This
document is the prose companion: the object model, the constraints JSON Schema can't fully express,
and the consumption contract. When the two disagree, the JSON Schema wins for field shape; this doc
wins for *meaning and process*.

## Object model

```
envelope
├── version            # const 1
├── spec               # companion doc filename (TECHNICAL_SPEC.md)
└── constraints[]      # flat list
    └── constraint
        ├── key          # REQUIRED. stable, kebab-case, UNIQUE across the file
        ├── kind         # REQUIRED. hard | soft
        ├── category     # REQUIRED. free-form group: platform|language|datastore|compliance|…
        ├── statement    # REQUIRED. the constraint, precise enough to check against
        ├── rationale    # REQUIRED. the external force (hard) or the default's justification (soft)
        ├── owner        # REQUIRED. team/role accountable for it
        ├── escapeHatch  # REQUIRED iff kind=soft; FORBIDDEN iff kind=hard
        └── tracesTo?    # optional PR-* ids this constraint is motivated by
```

## Hard vs. soft — the load-bearing distinction

- **Hard** — non-negotiable, externally imposed. Downstream agents may **not** violate it. It has no
  escape hatch. Example: `lang-python` — "Python 3.12 for all services," because of shared internal
  libraries and team standard.
- **Soft** — a strong default, overridable **with documented justification**. It MUST carry an
  `escapeHatch` saying what a valid override looks like. Example: `db-postgres` — "Prefer Postgres
  unless a workload demands otherwise," escape hatch "Document the workload need and the chosen
  alternative in an ADR."

The split is the whole point: a downstream coding agent can consume hard constraints programmatically
and lint generated code/IaC against them, while treating soft constraints as defaults it may override
on the record — so settled decisions are never silently broken nor needlessly re-litigated.

## Constraints not enforceable in JSON Schema

A validator must check these in addition to the JSON Schema:

1. **`key` uniqueness** — every `constraint.key` is unique across the file. JSON Schema can't assert
   cross-item uniqueness on a nested property.
2. **Escape-hatch rules** — re-checked with a clear message: every soft constraint has an
   `escapeHatch`; no hard constraint has one. (The schema also enforces this via `if/then`; the
   validator restates it legibly.)
3. **Traceability existence** — a product spec is **required** (a sibling `PRODUCT_SPEC.md` or
   `--product-spec`); every `tracesTo` id must appear in it. Constraints are bounded by the product
   they serve, so they are never validated in a vacuum — a missing spec is a usage error, not a
   silent downgrade.

## Consumption contract (how downstream agents use this)

- The **build-plan** ticket field `constraintRefs` references `key`s here. Its validator checks every
  referenced key exists in this file — making constraint compliance mechanically enforceable.
- A future **lint-constraints** tool can read a ticket's `constraintRefs` and check generated
  code/IaC against the matching `statement`s (e.g. fail a build that introduces a non-Postgres
  datastore when `db-postgres` is in scope and no ADR override is recorded).
- The **acceptance-spec** and **technical-spec** matrices share the `PR-*` spine in `tracesTo`.

## Future: machine-enforced constraint linting (design note, not built)

Today constraints are **referenced**, not linted: a build ticket names the `constraintRefs` it must
honor, and an executor or reviewer is trusted to comply. To make compliance *mechanically enforced*
later, this is the additive design — recorded now so the schema choices above stay forward-compatible:

- **Add an optional `check` predicate** to a constraint — the machine-checkable form of its prose
  `statement`. For example:
  ```yaml
  - key: db-postgres
    statement: "Prefer PostgreSQL unless a workload demands otherwise."
    check: { type: forbidden-import, patterns: ["pymongo", "redis"] }
  - key: lang-python
    statement: "Python 3.12 for all backend services."
    check: { type: required-runtime, runtime: python, range: ">=3.12,<3.13" }
  ```
  Predicate types worth supporting: `forbidden-import`, `required-runtime`, `file-glob-absent`,
  `file-glob-present`, and an escape-hatch `command` (a shell predicate that must exit 0).
- **A `lint-constraints` tool** reads a build ticket's `constraintRefs`, pulls each referenced
  constraint, and runs its `check` against the working tree or diff — failing the build on a hard
  violation, or requiring a recorded ADR override for a soft one.
- **The honest limitation.** Only a *subset* of constraints carry a cheap mechanical predicate:
  `lang-python`, `db-postgres`, and topology rules do; `latency-checkout` (needs a load test) and
  `compliance-soc2` (needs human judgment) do **not**. Linting would cover the predicate-bearing
  subset and explicitly leave the rest to review. Because `check` is optional and additive, adding it
  later breaks nothing here.

## Validation

Validate before publishing the spec:

```sh
# resolves a sibling PRODUCT_SPEC.md automatically:
${CLAUDE_PLUGIN_ROOT}/bin/validate-constraints constraints.yaml
# or point at a product spec elsewhere (required if none sits beside the constraints file):
${CLAUDE_PLUGIN_ROOT}/bin/validate-constraints constraints.yaml --product-spec ../product/PRODUCT_SPEC.md
```

It runs the JSON Schema plus the non-schema checks above. The technical-spec-architect refuses to
hand over a constraints file that fails validation.

## Example

See `examples/constraints.example.yaml` for a complete, valid envelope exercising both kinds, the
escape-hatch rule, categories, and the `tracesTo` spine.
