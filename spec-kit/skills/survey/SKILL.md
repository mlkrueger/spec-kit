---
name: survey
description: "Run spec-kit's Phase-0 repo survey on its own: map an existing repo's stack, seams, test harness, conventions, ID namespace, inherited constraints, and regression surface into REPO_MAP.md — scoped to a feature (default) or the full repo. Use before specifying brownfield work, or to refresh a stale survey. Triggers: 'survey the repo', 'map this repo for spec-kit', 'refresh the repo map', '/spec-kit:survey'."
---

# Standalone repo survey (Phase 0)

Produces the brownfield survey artifact without running the rest of the chain. Apply
`${CLAUDE_PLUGIN_ROOT}/reference/brownfield.md` (the survey tiers and the `REPO_MAP.md` structure
live there); follow `${CLAUDE_PLUGIN_ROOT}/reference/single-phase.md` for the shared ceremony. This
phase is performed inline — there is no survey agent to launch.

## Procedure

1. **Pick the tier.**
   - **Scoped** (default): the user names a feature or change; survey only what it touches. Written
     to `features/<slug>/REPO_MAP.md` with an explicit **"not surveyed"** section.
   - **Full**: first spec-kit run in the repo, wide blast radius, or the user asks for the whole
     map. Written to repo-root `REPO_MAP.md`.
   - If a root `REPO_MAP.md` already exists, **delta off it**: verify/extend the sections in scope
     and refresh anything reality contradicts — don't re-survey from scratch.
2. **Survey with grounding.** Every claim comes from the repo itself (Glob/Grep/Read real files) —
   manifests and lockfiles for the stack (→ `reference/profiles/<stack>.md` keys), real paths for
   seams, actual test commands and naming infixes, the existing `PR-*`/key namespace from
   `features/*/` and prior plans. Capture every `REPO_MAP.md` section listed in `brownfield.md`,
   including the integration points + regression surface — split **must-stay-green** vs.
   **must-be-migrated** when the upcoming work changes existing behavior.
3. **Present** the survey for correction — this is the cheapest place to fix a wrong assumption
   about the user's repo. Ask them to confirm the seams, harness, conventions, and (change mode)
   the regression-surface split.
4. **Stop.** Suggest the natural next step (`/spec-kit:product-spec` with the survey, or
   `/spec-kit:run` for the full chain) — don't launch it unasked.
