---
name: challenge
description: "Adversarially challenge any spec-kit artifact (product spec, technical spec + constraints, design spec, acceptance plan, build plan) as a one-off: launch the spec-challenger agent for a single cold-read pass against the phase rubric and present its disposition report (reviews/CHALLENGE_<PHASE>.md). The challenger files findings, never edits the artifact. Useful for specs that predate spec-kit, re-challenging after manual edits, or a second opinion before a review. Triggers: 'challenge this spec', 'poke holes in the technical spec', 'red-team the build plan', '/spec-kit:challenge'."
---

# Single pass: adversarial challenge

Thin wrapper around the **spec-challenger** agent. Follow
`${CLAUDE_PLUGIN_ROOT}/reference/single-phase.md` for the shared ceremony (input resolution, mode
detection, present-and-stop); the rubrics and report format are in
`${CLAUDE_PLUGIN_ROOT}/reference/challenge-standards.md`.

**No loop.** This skill runs pass 1 only and presents the report — the revise-or-rebut loop
belongs to the orchestrating skills (`/spec-kit:run`, the per-phase skills). If the user wants the
architect to respond to the findings, point them at the matching per-phase skill (re-invoke the
architect with the report as feedback), then optionally re-run this skill on the revision.

## Procedure

1. **Resolve the target.** The artifact to challenge is the skill argument (a path, or a phase name
   like "the technical spec" resolved per single-phase input resolution). Identify which phase
   rubric applies from the artifact type; ask if genuinely ambiguous.
2. **Resolve its upstream contract.** Gather the same upstream inputs the authoring architect had
   (e.g. for a technical spec: `PRODUCT_SPEC.md` + `REPO_MAP.md`/repo in feature mode). If an
   upstream artifact is missing — common for hand-written specs that predate spec-kit — proceed,
   telling the challenger what's absent: it challenges internal coherence and repo reality, and
   flags the missing contract itself.
3. **Launch spec-challenger** (Agent tool) with the artifact, the upstream inputs, and the
   applicable rubric section. Never pass any authoring conversation or reasoning — the cold read
   is the point. It writes `reviews/CHALLENGE_<PHASE>.md` (feature mode:
   `features/<slug>/reviews/`) and does not touch the artifact.
4. **Present** the report: the verdict line first, then findings severity-ordered (`blocker` /
   `major` / `question`), then the certification note (what was attacked and held). Every finding
   already names its concrete failure scenario — surface those, not just the claims.
5. **Stop.** Suggest natural next steps — route the blockers/majors through the matching per-phase
   skill so the architect can revise or rebut, or take the findings to review as-is. Never edit
   the artifact yourself and never launch the architect unasked.
