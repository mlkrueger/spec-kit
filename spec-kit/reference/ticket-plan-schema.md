# Ticket Plan Schema (the neutral contract — LEGACY)

> **Legacy.** The standalone ticket plan was retired as a produced artifact when `test-spec` folded into
> `spec-kit`: its job is now split between the **build plan**'s inline `tddCases` (unit/integration) and
> the **acceptance plan** (E2E). No `spec-kit` agent produces a `ticket-plan.yaml`. This schema +
> `bin/validate-ticket-plan` are retained only so a **pre-existing** ticket plan can still be published
> by the publisher skills. For new work, use `build-plan-schema.md` and `acceptance-plan-schema.md`.
>
> Originally the tracker-neutral hand-off between the `test-spec-architect` agent (producer) and the
> tracker publisher skills (consumers). A publisher reads the plan plus its own per-team config and
> creates/updates real tickets.
>
> **The plan holds intent only.** It contains no tracker-specific fields. *What to test* lives in the spec; *which tickets* lives in the plan; *where and how they land* lives in per-team publisher config. Keep these three separate and every piece stays portable.

The machine-validatable definition is `ticket-plan.schema.json` (JSON Schema draft 2020-12). This document is the prose companion: the object model, the constraints JSON Schema can't express, the neutral→tracker mapping, and the idempotency convention. When the two disagree, the JSON Schema wins for field shape; this doc wins for *meaning and process*.

## Object model

```
plan
├── version            # const 1
├── spec               # companion doc filename (TEST_SPEC.md)
├── title
├── milestone          # exactly one top-level grouping
│   ├── key            # stable, kebab-case, unique
│   ├── name
│   └── description?
└── tickets[]          # flat list — hierarchy is expressed by relationships, not nesting
    └── ticket
        ├── key                 # REQUIRED. stable, kebab-case, UNIQUE across the plan
        ├── title               # REQUIRED
        ├── layer               # REQUIRED. unit|integration|component|e2e|refactor|harness|ci
        ├── stack?              # polyglot key (python|web|go-api|…); "-" or omit if single-stack
        ├── priority?           # low|medium|high|urgent
        ├── estimate?           # neutral points
        ├── labels?             # free-form neutral strings
        ├── description         # REQUIRED. markdown body
        ├── cases               # REQUIRED for test layers. "input -> expected", incl. edges + failure modes
        ├── modulesInScope      # REQUIRED. REAL verified paths
        ├── acceptanceCriteria  # REQUIRED. checkable DoD checklist
        ├── parent?             # relationship: key of a parent ticket
        └── blockedBy?          # relationship: keys of prerequisite tickets
```

### Why flat + relationships (not nested structure)

The plan is the **narrow waist**. It expresses *relationships* — `parent` ("belongs under") and `blockedBy` ("must come after") — by `key`. Each publisher (the adapter) translates those into its tracker's *native structure*, degrading gracefully where the tracker can't represent them. The plan never commits to a tracker-ism like "sub-task".

## Constraints not enforceable in JSON Schema

A validator must check these in addition to the JSON Schema:

1. **`key` uniqueness** — every `ticket.key` (and the `milestone.key`) is unique across the plan. JSON Schema can't assert cross-item uniqueness on a nested property.
2. **Referential integrity** — every `parent` and every `blockedBy` entry references a `key` that exists in `tickets[]`.
3. **No cycles** — `parent` and `blockedBy` graphs are acyclic.
4. **Real paths** — `modulesInScope` entries are real, verified paths in the repo (the architect verifies these before emitting; "the relevant files" is never acceptable). See `testing-standards.md` → *Ticket structure*.

## Neutral → tracker mapping

Adapters implement this table. Where a tracker lacks a concept, use the documented degradation.

| neutral | Linear | Jira | GitHub |
|---|---|---|---|
| `milestone` | Project or Milestone | Epic | Milestone |
| `ticket` | Issue | Story / Task | Issue |
| `layer`, `labels`, `stack` | labels | labels / component | labels |
| `priority` | priority | priority | label (`priority:high`) |
| `estimate` | estimate | story points | — (dropped) |
| `parent` | sub-issue | sub-task | checklist item in parent body + "part of #N" |
| `blockedBy` | "blocked by" relation | "is blocked by" link | "blocked by #N" in body |
| `description` + `cases` + `modulesInScope` + `acceptanceCriteria` | rendered issue body | rendered description | rendered body |
| `key` | idempotency marker (see below) | idempotency marker | idempotency marker |

Routing values that are **not** in the plan — which Linear team, which Jira project key, which issue type per layer, the priority-scale mapping, the label prefix, default assignee — live in per-team publisher config (`${CLAUDE_PROJECT_DIR}/.spec-kit/publisher.yaml`), filled in once at onboarding. The same plan maps onto any tracker by swapping config.

> **Note (post-retirement):** the publish *mechanics* below — stamping, the marker prefix, skip/update — are now governed by the shared `publishing.md` contract, which uses the `skp:` / `skp-key:` prefix (configurable). The `tsp:` examples below are historical; the active prefix is whatever `publishing.md` + your publisher config define.

## Idempotency: stamping

Re-publishing a plan must **update**, not duplicate. Re-runs are normal: the spec gets revised, a run fails partway, or a different teammate runs it. Idempotency means running twice leaves the tracker in the same state as running once.

The mechanism is **stamping**: the durable identity lives on the created ticket, in the tracker — not in a local file (which breaks across machines/people, exactly what a shared plugin can't rely on).

A publisher MUST:

1. **On create**, stamp the ticket's `key` onto the created issue in a machine-findable place:
   - preferred: a label, namespaced — `tsp:<key>` (e.g. `tsp:unit-recurrence`);
   - always also: an HTML-comment marker on its own line in the issue body — `<!-- tsp-key: <key> -->` — as a universal fallback for trackers/searches where label lookup is unavailable.
2. **Before create**, search the tracker for the marker for that `key`. If found, the ticket already exists:
   - **default (skip mode):** leave it untouched — never silently clobber manual edits.
   - **`--update` mode:** overwrite the publisher-managed fields (title, body, labels, relations) from the plan, preserving anything outside the managed markers.
3. Stamp the `milestone.key` the same way on the milestone/epic/project, so the grouping is also matched on re-run rather than recreated.

The marker prefix (`tsp:` / `tsp-key:`) is configurable per team but must be stable for a given tracker so future runs find prior tickets.

## Validation

Validate a plan before publishing:

```sh
# any JSON Schema validator that supports draft 2020-12, e.g.:
check-jsonschema --schemafile reference/ticket-plan.schema.json ticket-plan.yaml
```

then apply the four non-schema constraints above. A publisher should refuse to run on a plan that fails validation.

## Example

See `examples/ticket-plan.example.yaml` for a complete, valid plan exercising every field, both a `refactor` blocker and the tickets that depend on it, and a `parent`/sub-ticket relationship.
