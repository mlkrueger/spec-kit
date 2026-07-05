---
name: build-plan
description: "Run just the build-plan phase: decompose TECHNICAL_SPEC.md + constraints.yaml into a dependency-ordered, test-first BUILD_PLAN.md + validated build-plan.yaml via the build-plan-architect agent, then present and stop. Also the light-path entry for small changes (mini spec, no tech spec). Triggers: 'break this into tickets', 'make the build plan', '/spec-kit:build-plan'."
---

# Single phase: build plan

Thin wrapper around the **build-plan-architect** agent. Follow
`${CLAUDE_PLUGIN_ROOT}/reference/single-phase.md` for the shared ceremony.

## Procedure

1. **Resolve inputs.** `PRODUCT_SPEC.md` is required (it anchors `tracesTo`). `TECHNICAL_SPEC.md` +
   `constraints.yaml` are the normal primary inputs — but on brownfield's **light path** (small,
   single-seam change with no architectural decision) a mini product spec alone is legitimate:
   tickets then carry no `constraintRefs` and the validator needs no constraints file. If the
   technical spec is missing and the work *does* involve a load-bearing decision, offer
   `/spec-kit:technical-spec` first instead. In feature mode resolve the survey.
2. **Launch build-plan-architect** (Agent tool). It writes `BUILD_PLAN.md` + `build-plan.yaml` and
   self-validates (feature mode: verified real paths, regression clauses; change mode:
   test-migration tickets, the four-clause done-gate).
3. **Validate independently:**
   ```sh
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-build-plan build-plan.yaml --constraints constraints.yaml --product-spec PRODUCT_SPEC.md
   ```
   (Omit `--constraints` on the light path.) A failure means re-invoking the agent with the errors
   before presenting.
4. **Present** the dependency order, `tier` tags, inline `tddCases`, the done-gate ticket, and —
   feature mode — a grounding spot-check (sample `modulesInScope` paths and confirm they exist).
   Cross-check that every `PR-*` is covered by at least one ticket's `tracesTo`.
5. **Warn + stop.** If no acceptance plan exists, note the done-gate references one — suggest
   `/spec-kit:acceptance-plan` (unless the change alters no user-observable journey). Point to
   execution + the publisher skills; don't launch anything unasked.
