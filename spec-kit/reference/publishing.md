# Publishing Contract (shared by every spec-kit publisher)

> The tracker-neutral plans `spec-kit` emits — `build-plan.yaml` and `acceptance-plan.yaml` — are
> published to a tracker by a **publisher skill** (`publish-linear`, `publish-jira`, …). This document
> is the **shared contract** every publisher obeys: plan-kind detection, body rendering, idempotency
> via key stamping, the neutral→tracker degradation principle, and config. A publisher's own `SKILL.md`
> adds only the tracker-specific mapping and the MCP calls — everything portable lives here, so the
> publishers stay identical except where the trackers genuinely differ.

## What a publisher does (and does not)

A publisher reads a **validated** neutral plan plus its **per-team config**, and creates/updates real
issues. It maps *intent* (the plan) onto a *tracker* (via config). It never invents routing the plan
deliberately omits (which team, which project, which issue type, the priority scale) — that is config.

## Step 0 — detect the plan kind and validate

Pick the plan kind and its validator before anything else; refuse to publish a plan that fails:

| Plan file | Kind | Validator |
|---|---|---|
| `build-plan.yaml` (or tickets with `tddCases`/`interface`/`constraintRefs`, layers `feature`/`interface`/`migration`/…) | build plan | `${CLAUDE_PLUGIN_ROOT}/bin/validate-build-plan <plan> --constraints <constraints.yaml> --product-spec <PRODUCT_SPEC.md>` |
| `acceptance-plan.yaml` (or tickets with layers `e2e`/`harness`) | acceptance plan | `${CLAUDE_PLUGIN_ROOT}/bin/validate-acceptance-plan <plan> --product-spec <PRODUCT_SPEC.md>` |
| `ticket-plan.yaml` (legacy; tickets with test-pyramid layers `unit`/`integration`/`component`/`e2e`) | ticket plan (legacy) | `${CLAUDE_PLUGIN_ROOT}/bin/validate-ticket-plan <plan>` |

Resolve `--constraints` from the plan's top-level `constraints:` field (or a sibling `constraints.yaml`)
and `--product-spec` from a sibling `PRODUCT_SPEC.md`. All three plan kinds share the same milestone +
flat-tickets + `key`/`parent`/`blockedBy` shape, so everything below is plan-kind-agnostic except body
rendering.

*(The legacy `ticket-plan` is supported for publishing a **pre-existing** plan only; no `spec-kit` agent
produces one — see `ticket-plan-schema.md`. For its body, render `description` + `## Cases` (`cases`) +
the common sections.)*

## Step 1 — resolve config

Read `${CLAUDE_PROJECT_DIR}/.spec-kit/publisher.yaml`. If absent, walk the user through creating it
(ask at least the tracker's team/project equivalents) — never guess routing. See *Config* below.

## Step 2 — resolve the milestone (the container)

Search the tracker for an existing container stamped with the plan's `milestone.key` (the namespaced
label or the body marker). Create it if absent, **stamping the key**. This is the
project/epic/milestone all issues live under.

## Step 3 — preview & confirm (always)

Build the full create/update set in memory and show the user a summary: N to create, M to update, K to
skip, plus the container. **Get explicit confirmation before any write.** `--dry-run` stops here; and
the first pass should always preview regardless. Never create issues without confirmation.

## Step 4 — create/update issues, idempotently, in dependency order

For each ticket, ordering so a ticket's `blockedBy`/`parent` targets are created first:

1. **Search** for an existing issue stamped with this ticket's `key` (namespaced label first, body
   marker as fallback).
2. **Found + skip mode (default):** leave it untouched; capture its id for relation wiring. **Found +
   `--update`:** overwrite only the publisher-managed fields. **Not found:** create it.
3. **Render the body** (below) and **stamp** the `key` (namespaced label on create + the hidden body
   marker line).

## Step 5 — wire relationships, then report

Once every issue exists and you hold their ids: set `parent` (sub-issue/sub-task/checklist per tracker)
and `blockedBy` (a "blocked by" relation/link, or text where unsupported). Then report created /
updated / skipped issues with their tracker identifiers/URLs.

## Body rendering (which sections, by plan kind)

Render the issue body from the ticket's fields, in this order, omitting any absent field. The hidden
marker line always comes last.

**Common (both kinds):** the `description` markdown, then —
- `## Modules in scope` ← `modulesInScope`
- `## Traces to` ← `tracesTo` (the `PR-*` IDs)
- `## Acceptance criteria` ← `acceptanceCriteria` (render as a checklist)
- `<!-- <markerPrefix>-key: <key> -->` (hidden marker, always last)

**Build-plan tickets add, before Modules in scope:**
- `## Interface` ← `interface`
- `## TDD cases (write first)` ← `tddCases`
- `## Constraints to honor` ← `constraintRefs`

**Acceptance-plan tickets add, before Modules in scope:**
- `## Journeys` ← `cases` (the Given/When/Then list)

## Label mapping (neutral → tracker labels)

These neutral fields become labels (a publisher may prefix/translate per config):
- `layer`, `stack`, and any free-form `labels`.
- `tier` (build-plan) → a label like `tier:simple` so an execution orchestrator can query routable work.
- each `tracesTo` `PR-*` → a label like `pr:PR-checkout-guest`, so coverage is queryable in the tracker.

`priority` maps via `config.priorityMap`; `estimate` maps to the tracker's points field (dropped where
unsupported). Everything not mapped here stays out of the tracker.

## Idempotency: stamping

Re-publishing must **update, not duplicate**. The durable identity lives on the created issue (in the
tracker), never in a local file — so re-runs from any machine/person converge. A publisher MUST:

1. **On create**, stamp the ticket's `key` in two machine-findable places: a namespaced label
   `<markerPrefix>:<key>` and a hidden body marker `<!-- <markerPrefix>-key: <key> -->`.
2. **Before create**, search for that marker. If found: **skip** (default — never clobber manual edits)
   or, with `--update`, overwrite only the managed body sections and managed labels/relations, leaving
   human-added comments/labels/state alone.
3. Stamp `milestone.key` the same way so the container is matched on re-run, not recreated.

`markerPrefix` defaults to `skp` (spec-kit plan); it is configurable per team but must stay **stable**
for a given tracker so future runs find prior issues.

## Config

`${CLAUDE_PROJECT_DIR}/.spec-kit/publisher.yaml` holds the routing each plan deliberately omits. The
tracker-specific keys live in each publisher's `SKILL.md`; the shared shape is:

```yaml
tracker: linear            # or: jira
markerPrefix: "skp"        # idempotency label/marker prefix — keep stable forever
labelPrefix: ""            # optional prefix applied to neutral labels
priorityMap:               # neutral priority -> the tracker's scale (see the publisher's SKILL.md)
  urgent: ...
  high: ...
  medium: ...
  low: ...
# ...plus the tracker-specific routing block (linear: / jira:) from the publisher SKILL.md
```

The same plan publishes to any tracker by swapping this config — the plan and the architects never
change.
