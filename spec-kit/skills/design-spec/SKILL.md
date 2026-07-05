---
name: design-spec
description: "Run just the design phase: turn an approved PRODUCT_SPEC.md into the visual + interaction contract — STYLE_TILE.md (tone sign-off first), UI_STYLE_GUIDE.md, and a validated design-tokens.yaml — via the design-spec-architect agent. Reference-controlled when a brand guide / existing design system / reference repo or site exists (extraction with provenance; the reference wins); invention mode otherwise. Frontend features only. Triggers: 'design the look and feel', 'extract our brand into tokens', 'make it match our brand guide / site X', '/spec-kit:design-spec'."
---

# Single phase: design spec

Thin wrapper around the **design-spec-architect** agent. Follow
`${CLAUDE_PLUGIN_ROOT}/reference/single-phase.md` for the shared ceremony.

## Procedure

1. **Resolve inputs.** `PRODUCT_SPEC.md` is required (personas, brand intent, UX states); if
   missing, offer `/spec-kit:product-spec` first. If the feature has **no UX surface**, say so and
   stop — this phase is frontend-only. **Gather the references**: ask for / locate any brand guide
   (PDF/markdown/URL), the design system already in the repo, a designated reference repo, or
   reference sites/screenshots — their presence decides the mode (reference-controlled vs.
   invention), and precedence follows `reference/design-spec-standards.md`.
2. **Launch design-spec-architect** (Agent tool) with the spec + references. It produces the tile
   first, then the guide + tokens.
3. **Checkpoint the tile before the system.** Present `STYLE_TILE.md` for approval — in reference
   mode the question is "did we extract your brand correctly?", in invention mode "is this the
   right vibe?" If the tone is wrong, loop here — it's cheap at the tile, expensive after. Then
   present `UI_STYLE_GUIDE.md` + `design-tokens.yaml`.
4. **Validate independently:**
   ```sh
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-design-tokens design-tokens.yaml --product-spec PRODUCT_SPEC.md
   ```
   A failure means re-invoking the agent with the errors before presenting. Surface any flagged
   **accessibility deviations** (reference pairings that fail AA + proposed accessible variants) —
   that decision belongs to the user, never made silently.
5. **Warn + stop.** If a `build-plan.yaml` predates these artifacts, note its frontend tickets may
   be stale (they should consume the tokens/guide). Suggest `/spec-kit:build-plan` next; don't
   launch it unasked.
