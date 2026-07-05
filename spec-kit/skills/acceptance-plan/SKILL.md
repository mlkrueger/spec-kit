---
name: acceptance-plan
description: "Run just the acceptance phase: turn PRODUCT_SPEC.md's acceptance criteria into ACCEPTANCE_SPEC.md + a validated acceptance-plan.yaml of E2E journeys via the acceptance-spec-architect agent, then present and stop. Triggers: 'write the acceptance tests', 'define the E2E journeys', 'what makes this done', '/spec-kit:acceptance-plan'."
---

# Single phase: acceptance plan

Thin wrapper around the **acceptance-spec-architect** agent. Follow
`${CLAUDE_PLUGIN_ROOT}/reference/single-phase.md` for the shared ceremony.

## Procedure

1. **Resolve inputs.** `PRODUCT_SPEC.md` is required; if missing, offer `/spec-kit:product-spec`
   first. In feature mode resolve the survey (the existing E2E harness to reuse lives there). The
   technical spec is optional context (harness/runner only).
2. **Launch acceptance-spec-architect** (Agent tool). It writes `ACCEPTANCE_SPEC.md` +
   `acceptance-plan.yaml` and self-validates (feature mode: existing harness reused, must-stay-green
   journeys named; change mode: superseded journeys + replacements).
3. **Validate independently:**
   ```sh
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-acceptance-plan acceptance-plan.yaml --product-spec PRODUCT_SPEC.md
   ```
   A failure means re-invoking the agent with the errors before presenting.
4. **Present** the journeys (Given/When/Then), what each proves, the harness ticket, what was
   deliberately pushed to lower layers, and — feature mode — the regression-surface journeys.
   Confirm every acceptance criterion in the product spec is covered by at least one journey.
5. **Warn + stop.** Note the sibling `build-plan.yaml`'s done-gate references this plan; if the
   build plan predates this regeneration, it may be stale. Suggest `/spec-kit:build-plan` or the
   publisher skills next; don't launch them unasked.
