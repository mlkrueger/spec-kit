---
name: ci
description: "Run the CI/CD phase as a one-off via the ci-architect agent: DESIGN a pipeline contract (CI_SPEC.md — gates, budgets, required checks, environments, secrets, rollback) from the specs/plans, or AUDIT an existing pipeline against the CI standards (CI_AUDIT.md + optional remediation build-plan.yaml). Triggers: 'design our CI', 'audit the pipeline', 'is CI actually enforcing our gates', '/spec-kit:ci [design|audit]'."
---

# CI/CD: design or audit

Thin wrapper around the **ci-architect** agent. Follow
`${CLAUDE_PLUGIN_ROOT}/reference/single-phase.md` for the shared ceremony.

## Procedure

1. **Resolve mode and inputs.** Mode is the skill argument (`design` | `audit`); if absent, infer —
   real workflow files in the repo ⇒ audit (confirm), none ⇒ design. Design mode wants
   `TECHNICAL_SPEC.md` + `constraints.yaml` and any plans (resolve per the shared order); it can run
   from the repo alone, but note which bars come from specs vs. inference. Audit mode wants the
   workflows plus whatever specs/plans define the bars; pass forge access (`gh`/`glab`) context if
   available so enforcement can be verified rather than inferred. Ask whether the user wants the
   remediation plan with an audit, or the audit alone first.
2. **Launch ci-architect** (Agent tool) with the mode and inputs. It writes `CI_SPEC.md` (design) or
   `CI_AUDIT.md` (+ optional `ci-remediation.build-plan.yaml`) per `reference/ci-standards.md`.
3. **Validate independently** when a remediation plan was emitted:
   ```sh
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-build-plan ci-remediation.build-plan.yaml
   ```
   A failure means re-invoking the agent with the errors before presenting.
4. **Present.** Design: the gates-to-bars table, budgets, required checks, environments/promotion,
   and the rollback story. Audit: the **bar-diff table first** (spec'd bars vs. wired-and-blocking),
   then findings by severity — anything `high` (unguarded merge path, secrets exposure) leads.
5. **Stop.** Design hands off to `/spec-kit:build-plan` (the `ci` tickets); audit hands off to the
   remediation plan and the publishers. Suggest, don't launch.
