---
name: run
description: "Drive the full spec-phase chain for a feature: product spec -> technical spec + constraints -> acceptance plan + build plan, invoking each spec-kit architect agent in order and STOPPING for your approval between phases (including the principal-eng review between the technical spec and the build plan). Each artifact is adversarially challenged by the spec-challenger agent before its checkpoint, so your review starts from the contested points. Works greenfield (empty repo) or feature-addition (brownfield: surveys the existing repo first, scopes the feature, inherits existing conventions as constraints). For incremental tweaks, invoke a single architect agent directly instead. Triggers: 'do a full spec run', 'run spec-kit', 'spec this feature', 'add this feature to the repo', '/spec-kit:run'."
---

# Full spec-phase run

Orchestrates the `spec-kit` architect agents (product, technical, design for UI features,
acceptance, build) into one chain, with a **human-approval checkpoint
between every phase**. Each phase produces a validated artifact that is the contract for the next; you
review and approve before the chain proceeds. The skill never runs the whole chain unattended — the
checkpoints are the point.

```
intake ─▶ [0] (feature mode only) repo survey ─▶ REPO_MAP.md
                 ⏸ checkpoint: confirm the survey reflects the repo
        ─▶ [1] product-spec-architect ─▶ PRODUCT_SPEC.md
                 ⏸ checkpoint: approve the product spec (and its PR-* IDs)
        ─▶ [2] technical-spec-architect ─▶ TECHNICAL_SPEC.md + constraints.yaml
            ├─ design-spec-architect  ─▶ STYLE_TILE.md, UI_STYLE_GUIDE.md,      (in parallel;
            │                             design-tokens.yaml                     UI features only)
                 ⏸ checkpoint: PRINCIPAL/STAFF-ENG REVIEW — architecture + constraints
                 ⏸ + DESIGN REVIEW — tile first, then guide + tokens (UI features only)
        ─▶ [3] acceptance-spec-architect + build-plan-architect  (run in parallel)
                 ⏸ checkpoint: approve the acceptance plan + build plan
        ─▶ hand-off: execution + publishing (+ /spec-kit:ci when the pipeline needs work)
```

Between each phase's validator and its ⏸ checkpoint sits an **adversarial challenge step** (the
**spec-challenger** agent) — see "The challenge step" below. It hardens the draft and annotates
what remains genuinely disputed, so each checkpoint presents the artifact *plus* a disposition
report, leading with the contested points.

## Mode

Determine the mode at the start; it shapes the whole run:

- **greenfield** — empty/new repo. Skip Phase 0. Artifacts go where the user wants (repo root by
  default). IDs are plain `PR-<req>`.
- **feature** (brownfield) — working in a pre-existing repo. **Run Phase 0 first**, apply
  `${CLAUDE_PLUGIN_ROOT}/reference/brownfield.md` throughout, write artifacts under
  `features/<feature-slug>/`, and use feature-prefixed IDs (`PR-<feature>-<req>`). Pick this mode when
  the working directory already contains a real codebase, or the user says "add … to the repo."
  Within feature mode, note per-requirement whether the work **adds** behavior or **changes** existing
  behavior — changed requirements get brownfield's *change mode* (behavior delta, `Modifies:` links,
  the regression-surface split, test-migration tickets) threaded through every phase below.

If unsure which mode, look: a populated repo with source + tests ⇒ feature; an empty/scaffold-only dir
⇒ greenfield. Confirm with the user when ambiguous.

**Light path.** For a bugfix-sized, single-seam change, offer brownfield's **light path** instead of
the full chain: inline scoped survey → mini product spec (1–3 `PR-*`) → build plan (skip the technical
spec when no architectural decision is being made), acceptance plan only if a user-observable journey
changes. One combined checkpoint (mini spec + plan) replaces the per-phase gates; validators still run.
**The challenge step is skipped on the light path** — a challenger on a bugfix-sized change is ceremony.

## Inputs

- **Brief:** the idea / problem statement / transcript / path, as the skill argument. If absent, ask
  the user for it before starting.
- **Feature slug:** in feature mode, derive a short kebab-case slug for the feature (e.g.
  `guest-checkout`) — it names the `features/<slug>/` dir and prefixes all IDs/keys. Confirm it with the
  user.
- **Resume:** if some artifacts already exist (for feature mode, under `features/<slug>/`), detect them
  and offer to **start from the first missing phase** rather than regenerating — confirm which phase to
  start at. Still honor every downstream checkpoint.

## The challenge step (between validator and every ⏸)

After a phase's artifact validates and before its checkpoint, launch the **spec-challenger** agent
(Agent tool) per `${CLAUDE_PLUGIN_ROOT}/reference/challenge-standards.md`. Give it the artifact and
the same upstream inputs the architect had — **never the architect's conversation or reasoning**
(cold read is the point). It files a disposition report at `reviews/CHALLENGE_<PHASE>.md`
(feature mode: `features/<slug>/reviews/`) and never edits the artifact.

**The loop (this skill mediates; the agents never converse directly):**

1. Challenger pass 1 → report with findings `open` (or an explicit certification).
2. Route `blocker`/`major` findings to the architect (re-invoke with the report). `question`s skip
   the loop — they ride the report straight to the human.
3. The architect **revises or rebuts** per finding — it is never obligated to comply; rebuttals are
   recorded verbatim in the report.
4. Challenger pass 2, **scope-locked** (prior blockers/majors + revised sections only, no new
   findings) → each finding marked `resolved` / `rebutted — stands` / `open — contested`.
5. If the artifact changed, **re-run the phase validator** before presenting.

**Per-phase defaults** (the user can widen or narrow per run):

- **Product spec, technical spec (+ constraints):** full loop (2 passes) — errors there propagate
  furthest, and the principal-eng review is the gate the report most improves.
- **Design spec, acceptance plan, build plan:** single pass — challenge → architect responds →
  checkpoint, no pass 2. Already backstopped by validators, the coverage cross-check, and (design)
  the tile-first loop.
- **Light path:** off.

At the checkpoint, present the disposition report alongside the artifact, **leading with anything
`open — contested`** — the human adjudicates those; this skill never breaks ties. The report is
presentation + audit trail only: it is **never passed as input to downstream architects**.

## Procedure

Run phases in order. At each **⏸ checkpoint**, present the artifact(s) and the validator result, then
**stop and wait for explicit approval** before continuing. If the user requests changes, re-invoke that
phase's agent with the feedback and re-validate before re-presenting. Never skip a checkpoint.

### Phase 0 — Repo survey (feature mode only; skip for greenfield)
1. Apply `${CLAUDE_PLUGIN_ROOT}/reference/brownfield.md`. **Pick the survey tier:** a **scoped survey**
   (the default — only the seams, conventions, tests, and namespace the feature touches, written to
   `features/<feature-slug>/REPO_MAP.md` with an explicit "not surveyed" note) or a **full survey**
   (first spec-kit run in the repo, or wide-blast-radius work — the complete map at repo-root
   `REPO_MAP.md`). If a root `REPO_MAP.md` already exists, **delta off it**: read it, verify/extend
   only the sections this feature touches, and refresh anything reality contradicts. Either tier
   captures, per that doc: the stack(s) (→ `profiles/` keys), the architecture & module seams the
   feature attaches to, the test harness (commands, the **existing E2E harness**, naming infixes), the
   conventions, the existing feature areas, the **existing `PR-*`/ticket-key namespace**, the
   **inherited constraints**, and the integration points + regression surface — **split into
   must-stay-green vs. must-be-migrated when behavior is changing**.
2. **⏸ Checkpoint.** Present the survey and have the user **correct your understanding of their repo**
   — this is the cheapest place to fix a wrong assumption, before it propagates through every phase.
   Confirm the feature slug, the survey tier, and (change mode) the regression-surface split. **Get
   approval before phase 1.** Pass the survey to every later phase.

> **Feature mode paths.** All artifacts below live under `features/<feature-slug>/`; pass `REPO_MAP.md`
> to every agent; IDs/keys are feature-prefixed. The validator commands shown use bare filenames — in
> feature mode run them from inside `features/<feature-slug>/` (or prefix the paths) so the sibling
> `PRODUCT_SPEC.md`/`constraints.yaml` resolve.

### Phase 1 — Product spec
1. Launch the **product-spec-architect** agent (Agent tool) with the brief (and `REPO_MAP.md` in feature
   mode). It asks its own clarifying questions if the brief is thin; let it.
2. It writes `PRODUCT_SPEC.md` with `PR-*` requirement IDs (feature-prefixed in feature mode).
3. **Challenge (full loop).** Run the challenge step on `PRODUCT_SPEC.md` against the product-spec
   rubric — solutions in disguise, untestable criteria, missing non-goals, conflicting `PR-*`s.
4. **⏸ Checkpoint.** Show the user the product spec — especially the requirement list, non-goals, and
   the user-facing NFRs. Confirm the `PR-*` set is right; these IDs are the spine everything downstream
   traces to. **Get approval before phase 2.**

### Phase 2 — Technical spec + constraints ∥ design spec (UI features only)
1. Determine whether the feature **has a UI**: the product spec's UX-flows/states and personas make
   this obvious; confirm with the user when ambiguous. No UX surface ⇒ skip the design track
   entirely.
2. Launch the **technical-spec-architect** agent with `PRODUCT_SPEC.md` (and `REPO_MAP.md` + the
   codebase in feature mode, so it inherits existing conventions as hard constraints and designs to
   existing seams). **In parallel** (same message, two Agent calls) for UI features: the
   **design-spec-architect** with `PRODUCT_SPEC.md` + any design references (brand guide, existing
   design system, reference repo/sites — gather these at intake; their presence puts it in
   reference-controlled mode, where the reference wins).
3. They write `TECHNICAL_SPEC.md` + `constraints.yaml` (self-validated) and, for UI features,
   `STYLE_TILE.md` + `UI_STYLE_GUIDE.md` + `design-tokens.yaml`. Independently confirm:
   ```sh
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-constraints constraints.yaml --product-spec PRODUCT_SPEC.md
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-design-tokens design-tokens.yaml --product-spec PRODUCT_SPEC.md
   ```
4. **Challenge.** Run the challenge step on `TECHNICAL_SPEC.md` + `constraints.yaml` (**full
   loop**) — strawman alternatives, constraint-envelope gaps, NFR translations that don't follow,
   repo contradictions — and, for UI features, on the design artifacts (**single pass**) — UX
   states with no component, silently inherited accessibility deviations.
5. **⏸ Checkpoint — the principal/staff-eng review.** This is the load-bearing human gate. Present the
   architecture, the hard/soft constraint envelope (owners + escape hatches), the ADR-lite decisions
   and their rejected alternatives, and the NFR→numeric translations. Surface anything risky. The
   constraint envelope is the only place platform/language/compliance is decided, so make sure it's
   right — downstream agents treat it as given.
6. **⏸ Checkpoint — the design review (UI features only).** Present the **tile first** for tone
   sign-off (reference mode: "did we extract your brand correctly?"), then the guide + tokens with
   the validator result, and any flagged **accessibility deviations** in the reference (the user
   decides — never silently fixed, never silently shipped). A tone change loops at the tile before
   the systematic work is redone. **Get approval on both tracks before phase 3.**

### Phase 3 — Acceptance plan + build plan (parallel)
1. Launch **both** agents concurrently (two Agent tool calls in one message), since both consume the
   phase-2 outputs (pass `REPO_MAP.md` to both in feature mode):
   - **acceptance-spec-architect** with `PRODUCT_SPEC.md` → `ACCEPTANCE_SPEC.md` + `acceptance-plan.yaml`.
     In feature mode it reuses the existing E2E harness and names the regression-surface journeys.
   - **build-plan-architect** with `TECHNICAL_SPEC.md` + `constraints.yaml` + `PRODUCT_SPEC.md` (+
     `design-tokens.yaml` + `UI_STYLE_GUIDE.md` for UI features) → `BUILD_PLAN.md` + `build-plan.yaml`.
     In feature mode its `modulesInScope` reference real existing files to edit, and the done-gate adds
     the regression clause. For UI features it emits the theme-generation + `/style-tile` reference-page
     ticket that frontend tickets are `blockedBy`.
2. Each self-validates; independently confirm:
   ```sh
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-acceptance-plan acceptance-plan.yaml --product-spec PRODUCT_SPEC.md
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-build-plan build-plan.yaml --constraints constraints.yaml --product-spec PRODUCT_SPEC.md
   ```
3. **Challenge (single pass, both plans).** Run the challenge step — semantic traceability (does
   the journey actually *verify* the `PR-*` it cites), missing failure-path journeys, wrong
   dependency edges, `tddCases` that test implementation, `simple`-tier tickets that aren't,
   greenfield-smelling `modulesInScope`.
4. **⏸ Checkpoint.** Present both plans together: the build plan's dependency order, `tier` tags, and
   inline `tddCases`; the acceptance plan's journeys. Run the coverage cross-check below. **Get approval
   before hand-off.**

### Coverage cross-check (run before the final hand-off)
Confirm the traceability spine is complete across the chain:
- Every `PR-*` defined in `PRODUCT_SPEC.md` is covered by **at least one** `build-plan` ticket's
  `tracesTo` **and** at least one `acceptance-plan` journey's `tracesTo`. List any requirement missing
  design, build, or acceptance coverage — that gap is the most valuable thing this checkpoint surfaces.
- The build plan emitted its `ci` done-gate ticket (build suite ∧ acceptance plan both green).
- **Feature mode:** the done-gate also carries the **regression clause** (the must-stay-green set:
  pre-existing suite + existing E2E journeys outside the change), and every integration-touching build
  ticket asserts "existing tests covering <area> still pass." Confirm the feature's IDs/keys don't
  collide with the survey's namespace.
- **Grounding spot-check (feature mode):** sample a few `modulesInScope` paths from each plan and
  confirm they exist in the repo, and that named conventions (test-file infix, module seams) match
  reality — greenfield-ish output (invented paths, generic conventions) is the failure mode this
  catches.
- **Change mode:** every `Modifies:` requirement has a behavior delta in the product spec, a
  migration/rollout decision in the technical spec, **test-migration tickets** in the build plan for
  the must-be-migrated set, and updated journeys replacing any superseded ones in the acceptance plan.
- **UI features:** every UX state in `PRODUCT_SPEC.md` has a component in `UI_STYLE_GUIDE.md` to
  render it; `validate-design-tokens` passes (AA pairs hold); the build plan emitted the
  theme-generation/`/style-tile` ticket and its frontend tickets cite the guide's contracts.

### Hand-off
Summarize the artifacts produced and point to the next steps:
- **Execution:** an execution agent walks the build-plan tickets in dependency order (red → green →
  refactor), routing `simple`-tier tickets to a cheap model per the team's tier config. The feature is
  done when the build tickets are green **and** the acceptance plan is green. First-class companion
  (separate plugin, if installed): publish the plans, then `/dev-orchestrator:orchestrate <milestone>` —
  spec-kit's `tier` labels and criteria-bearing descriptions are exactly what it routes and gates on.
- **CI:** if the repo has no pipeline (or an unaudited one), point at `/spec-kit:ci` — design the
  pipeline contract (`CI_SPEC.md`) so the build plan's `ci` tickets wire designed gates, or audit the
  existing pipeline against the done-gate this run just defined.
- **Publishing:** run the tracker publisher skill to turn the neutral plans into issues.

## Checkpoint discipline

- **Never proceed past a ⏸ without explicit user approval.** "Looks good," "approved," "continue" — wait
  for it. Silence is not approval.
- **A failed validator blocks the checkpoint.** If an artifact doesn't validate, fix it (re-invoke the
  agent with the error) before presenting — never ask the user to approve an invalid artifact.
- **Contested findings lead the presentation.** When the challenge step ran, present the disposition
  report with the artifact, `open — contested` items first — those are the human's ruling to make.
  Never suppress a contested finding to smooth an approval, and never present challenger opinions as
  the architect's.
- **Changes loop, don't skip.** If the user wants edits at a checkpoint, re-run that phase's agent with
  the feedback, re-validate, and re-present the *same* checkpoint. Downstream phases stay gated until
  the upstream artifact is approved, because they consume it.
- **One phase at a time.** Do not pre-run a later phase "to save time" — a later phase built on an
  unapproved artifact is wasted if the user changes the upstream one.

## Single-phase / resume use

This skill is the *full* run. For incremental work, each phase has its own skill —
`/spec-kit:survey`, `/spec-kit:product-spec`, `/spec-kit:technical-spec`, `/spec-kit:acceptance-plan`,
`/spec-kit:build-plan` — which resolves inputs, invokes the same architect agent, validates, presents,
and stops (shared ceremony: `${CLAUDE_PLUGIN_ROOT}/reference/single-phase.md`). When resuming
mid-chain, start at the first missing/changed artifact and still honor every downstream checkpoint.
