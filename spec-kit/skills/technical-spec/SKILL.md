---
name: technical-spec
description: "Run just the technical-spec phase: turn an approved PRODUCT_SPEC.md into TECHNICAL_SPEC.md + a validated constraints.yaml via the technical-spec-architect agent, then present it for principal-eng review and stop. Triggers: 'design how we build this', 'write the tech spec', 'pin down the constraints', '/spec-kit:technical-spec'."
---

# Single phase: technical spec + constraints

Thin wrapper around the **technical-spec-architect** agent. Follow
`${CLAUDE_PLUGIN_ROOT}/reference/single-phase.md` for the shared ceremony.

## Procedure

1. **Resolve inputs.** `PRODUCT_SPEC.md` is required — resolve per the shared order (argument >
   feature dir > cwd); if missing, offer `/spec-kit:product-spec` first. In feature mode also
   resolve the survey and pass the codebase context.
2. **Launch technical-spec-architect** (Agent tool). It writes `TECHNICAL_SPEC.md` +
   `constraints.yaml` and self-validates (feature mode: inherited constraints folded in as hard;
   change mode: migration/rollout ADRs).
3. **Validate independently:**
   ```sh
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-constraints constraints.yaml --product-spec PRODUCT_SPEC.md
   ```
   A failure means re-invoking the agent with the errors before presenting.
4. **Present as the principal/staff-eng review** — the same load-bearing gate as `/spec-kit:run`'s
   phase-2 checkpoint: the architecture, the hard/soft constraint envelope (owners + escape
   hatches), the ADR-lite decisions and rejected alternatives, the NFR→numeric translations, and
   the traceability matrix. Surface anything risky; the envelope is the only place
   platform/language/compliance is decided.
5. **Warn + stop.** If plans exist downstream, note they may now be stale (`constraintRefs` and
   decomposition both hang off this spec). Suggest `/spec-kit:acceptance-plan` +
   `/spec-kit:build-plan` next; don't launch them unasked.
