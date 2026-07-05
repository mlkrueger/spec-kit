---
name: product-spec
description: "Run just the product-spec phase: turn an idea/brief/transcript into PRODUCT_SPEC.md (observable behaviors with PR-* IDs) via the product-spec-architect agent, then present it for review and stop. Works greenfield or brownfield (including change mode and mini specs for the light path). Triggers: 'write a product spec', 'spec the what/why', '/spec-kit:product-spec'."
---

# Single phase: product spec

Thin wrapper around the **product-spec-architect** agent. Follow
`${CLAUDE_PLUGIN_ROOT}/reference/single-phase.md` for the shared ceremony (input resolution, mode
detection, present-and-stop).

## Procedure

1. **Resolve inputs.** The brief (idea / problem statement / transcript / path) is the skill
   argument; ask for it if absent. Detect mode; in feature mode resolve the survey (`REPO_MAP.md`)
   if one exists, confirm the feature slug, and offer `/spec-kit:survey` if there is no survey and
   the work is more than a small change.
2. **Scale to the ask.** For a bugfix-sized, single-seam change, tell the agent a **mini product
   spec** is appropriate (brownfield light path: problem, 1–3 `PR-*`, acceptance criteria — no
   personas/metrics ceremony).
3. **Launch product-spec-architect** (Agent tool) with the brief + survey. It asks its own
   clarifying questions if the brief is thin; let it. It writes `PRODUCT_SPEC.md` (feature mode:
   under `features/<slug>/`, feature-prefixed IDs; change mode: behavior deltas + `Modifies:`).
4. **Present** as `/spec-kit:run`'s phase-1 checkpoint does: the requirement list and its `PR-*`
   IDs (the spine everything downstream traces to), non-goals, user-facing NFRs, and — change
   mode — the behavior deltas. There is no validator for the product spec; the review is the gate.
5. **Warn + stop.** If downstream artifacts exist (technical spec, plans), note they may now be
   stale. Suggest `/spec-kit:technical-spec` next; don't launch it unasked.
