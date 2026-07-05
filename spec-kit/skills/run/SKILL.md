---
name: run
description: "Drive the full spec-phase chain for a feature: product spec -> technical spec + constraints -> acceptance plan + build plan, invoking each spec-kit architect agent in order and STOPPING for your approval between phases (including the principal-eng review between the technical spec and the build plan). Works greenfield (empty repo) or feature-addition (brownfield: surveys the existing repo first, scopes the feature, inherits existing conventions as constraints). For incremental tweaks, invoke a single architect agent directly instead. Triggers: 'do a full spec run', 'run spec-kit', 'spec this feature', 'add this feature to the repo', '/spec-kit:run'."
---

# Full spec-phase run

Orchestrates the four `spec-kit` architect agents into one chain, with a **human-approval checkpoint
between every phase**. Each phase produces a validated artifact that is the contract for the next; you
review and approve before the chain proceeds. The skill never runs the whole chain unattended — the
checkpoints are the point.

```
intake ─▶ [0] (feature mode only) repo survey ─▶ REPO_MAP.md
                 ⏸ checkpoint: confirm the survey reflects the repo
        ─▶ [1] product-spec-architect ─▶ PRODUCT_SPEC.md
                 ⏸ checkpoint: approve the product spec (and its PR-* IDs)
        ─▶ [2] technical-spec-architect ─▶ TECHNICAL_SPEC.md + constraints.yaml
                 ⏸ checkpoint: PRINCIPAL/STAFF-ENG REVIEW — approve architecture + constraints
        ─▶ [3] acceptance-spec-architect + build-plan-architect  (run in parallel)
                 ⏸ checkpoint: approve the acceptance plan + build plan
        ─▶ hand-off: execution + publishing
```

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

## Inputs

- **Brief:** the idea / problem statement / transcript / path, as the skill argument. If absent, ask
  the user for it before starting.
- **Feature slug:** in feature mode, derive a short kebab-case slug for the feature (e.g.
  `guest-checkout`) — it names the `features/<slug>/` dir and prefixes all IDs/keys. Confirm it with the
  user.
- **Resume:** if some artifacts already exist (for feature mode, under `features/<slug>/`), detect them
  and offer to **start from the first missing phase** rather than regenerating — confirm which phase to
  start at. Still honor every downstream checkpoint.

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
   — this is the cheapest place to fix a wrong assumption, before it propagates through four phases.
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
3. **⏸ Checkpoint.** Show the user the product spec — especially the requirement list, non-goals, and
   the user-facing NFRs. Confirm the `PR-*` set is right; these IDs are the spine everything downstream
   traces to. **Get approval before phase 2.**

### Phase 2 — Technical spec + constraints
1. Launch the **technical-spec-architect** agent with `PRODUCT_SPEC.md` (and `REPO_MAP.md` + the
   codebase in feature mode, so it inherits existing conventions as hard constraints and designs to
   existing seams).
2. It writes `TECHNICAL_SPEC.md` + `constraints.yaml` and self-validates. Independently confirm:
   ```sh
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-constraints constraints.yaml --product-spec PRODUCT_SPEC.md
   ```
3. **⏸ Checkpoint — the principal/staff-eng review.** This is the load-bearing human gate. Present the
   architecture, the hard/soft constraint envelope (owners + escape hatches), the ADR-lite decisions
   and their rejected alternatives, and the NFR→numeric translations. Surface anything risky. The
   constraint envelope is the only place platform/language/compliance is decided, so make sure it's
   right — downstream agents treat it as given. **Get approval before phase 3.**

### Phase 3 — Acceptance plan + build plan (parallel)
1. Launch **both** agents concurrently (two Agent tool calls in one message), since both consume the
   phase-2 outputs (pass `REPO_MAP.md` to both in feature mode):
   - **acceptance-spec-architect** with `PRODUCT_SPEC.md` → `ACCEPTANCE_SPEC.md` + `acceptance-plan.yaml`.
     In feature mode it reuses the existing E2E harness and names the regression-surface journeys.
   - **build-plan-architect** with `TECHNICAL_SPEC.md` + `constraints.yaml` + `PRODUCT_SPEC.md` →
     `BUILD_PLAN.md` + `build-plan.yaml`. In feature mode its `modulesInScope` reference real existing
     files to edit, and the done-gate adds the regression clause.
2. Each self-validates; independently confirm:
   ```sh
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-acceptance-plan acceptance-plan.yaml --product-spec PRODUCT_SPEC.md
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-build-plan build-plan.yaml --constraints constraints.yaml --product-spec PRODUCT_SPEC.md
   ```
3. **⏸ Checkpoint.** Present both plans together: the build plan's dependency order, `tier` tags, and
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

### Hand-off
Summarize the artifacts produced and point to the next steps:
- **Execution:** an execution agent walks the build-plan tickets in dependency order (red → green →
  refactor), routing `simple`-tier tickets to a cheap model per the team's tier config. The feature is
  done when the build tickets are green **and** the acceptance plan is green.
- **Publishing:** run the tracker publisher skill to turn the neutral plans into issues.

## Checkpoint discipline

- **Never proceed past a ⏸ without explicit user approval.** "Looks good," "approved," "continue" — wait
  for it. Silence is not approval.
- **A failed validator blocks the checkpoint.** If an artifact doesn't validate, fix it (re-invoke the
  agent with the error) before presenting — never ask the user to approve an invalid artifact.
- **Changes loop, don't skip.** If the user wants edits at a checkpoint, re-run that phase's agent with
  the feedback, re-validate, and re-present the *same* checkpoint. Downstream phases stay gated until
  the upstream artifact is approved, because they consume it.
- **One phase at a time.** Do not pre-run a later phase "to save time" — a later phase built on an
  unapproved artifact is wasted if the user changes the upstream one.

## Single-phase / resume use

This skill is the *full* run. For incremental work, the user can invoke any architect agent directly
(e.g. just re-run `build-plan-architect` after a tech-spec tweak). When resuming mid-chain, start at the
first missing/changed artifact and still honor every downstream checkpoint.
