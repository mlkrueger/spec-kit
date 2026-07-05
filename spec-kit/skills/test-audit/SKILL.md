---
name: test-audit
description: "Audit an existing repo's test suites — scoped to unit, integration, e2e, or all layers — via the test-audit-architect agent: inventory + run the real suites, score them against the spec-kit testing standards, produce TEST_AUDIT.md, and optionally a validated test-backfill build-plan.yaml. Triggers: 'audit our tests', 'how good is our test coverage really', 'assess the e2e suite', 'plan test backfill', '/spec-kit:test-audit [unit|integration|e2e|all]'."
---

# Test audit

Thin wrapper around the **test-audit-architect** agent. Follow
`${CLAUDE_PLUGIN_ROOT}/reference/single-phase.md` for the shared ceremony; this phase needs no
upstream spec — the repo itself is the input.

## Procedure

1. **Resolve scope and inputs.** Scope is the skill argument: `unit`, `integration`, `e2e`, or
   `all` (default when absent). Pass any existing survey (`REPO_MAP.md`) and specs
   (`features/*/PRODUCT_SPEC.md`, `acceptance-plan.yaml`) — they give the audit a traceability spine.
   Ask whether the user wants the **backfill plan** emitted along with the audit, or the audit alone
   first.
2. **Launch test-audit-architect** (Agent tool) with the scope. It inventories and runs the real
   suites, scores the scoped layer(s) against the testing standards, and writes `TEST_AUDIT.md`
   (plus `test-backfill.build-plan.yaml` if requested).
3. **Validate independently** when a backfill plan was emitted:
   ```sh
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-build-plan test-backfill.build-plan.yaml
   ```
   (add `--product-spec` when its tickets trace to a spec). A failure means re-invoking the agent
   with the errors before presenting.
4. **Present** the scorecard and the findings ranked by severity — lead with anything `high` (a
   failure mode shipping undetected). Separate **real bugs** the audit surfaced from backfill work;
   bugs route to `/spec-kit:product-spec` or a bugfix light path, never hidden inside backfill.
5. **Stop.** Suggest next steps — emit/execute the backfill plan, publish it via
   `/spec-kit:publish-linear` / `/spec-kit:publish-jira` — but don't launch them unasked.
