# Single-Phase Invocation (Reference)

> The shared ceremony every per-phase skill (`/spec-kit:survey`, `/spec-kit:product-spec`,
> `/spec-kit:technical-spec`, `/spec-kit:acceptance-plan`, `/spec-kit:build-plan`, …) obeys. Each
> skill is a thin wrapper around one architect agent: resolve inputs, invoke, validate,
> **challenge**, present, **stop**. `/spec-kit:run` chains the same phases with checkpoints between them; a per-phase skill
> is one link of that chain, invoked on its own for incremental work. The wrappers differ only in
> which agent they launch and which artifacts they resolve — this document is the part they share.

## Input resolution

Resolve each required upstream artifact in this order; tell the user which was used:

1. **Explicit path** given as a skill argument — always wins.
2. **Feature directory** — when a feature slug is given or inferable (the user names the feature, or
   exactly one `features/<slug>/` contains the expected upstream artifacts), resolve inside
   `features/<slug>/`.
3. **Working directory / repo root** — a sibling `PRODUCT_SPEC.md`, `TECHNICAL_SPEC.md`, etc.

If a required upstream artifact is missing, **do not invent it and do not silently run the upstream
phase**. Say what's missing and offer the choice: run the upstream skill first, point at an existing
file elsewhere, or — for small changes — take brownfield's light path (a mini product spec is enough
to anchor `tracesTo`). If multiple `features/*/` candidates match and no slug was given, ask.

## Mode detection

Same posture as `/spec-kit:run`: a populated repo with source + tests ⇒ **feature mode** (apply
`brownfield.md`; artifacts under `features/<slug>/`; feature-prefixed IDs/keys); an empty or
scaffold-only directory ⇒ **greenfield**. Confirm when ambiguous. In feature mode, read the survey
(`REPO_MAP.md`, root or feature-scoped) if one exists and pass it to the agent. If none exists,
offer `/spec-kit:survey` first — or proceed and note that the agent must ground everything in the
repo directly (brownfield principle 5 applies regardless).

## Invoke, then validate

Launch the phase's architect agent (Agent tool) with the resolved inputs. When the phase has a
validator, run it independently after the agent finishes — the agents self-validate, but the skill
confirms:

```sh
${CLAUDE_PLUGIN_ROOT}/bin/validate-constraints      constraints.yaml      --product-spec PRODUCT_SPEC.md
${CLAUDE_PLUGIN_ROOT}/bin/validate-acceptance-plan  acceptance-plan.yaml  --product-spec PRODUCT_SPEC.md
${CLAUDE_PLUGIN_ROOT}/bin/validate-build-plan       build-plan.yaml       --constraints constraints.yaml --product-spec PRODUCT_SPEC.md
```

(In feature mode run from inside `features/<slug>/` or prefix the paths.) **A failed validator is
never presented as done** — re-invoke the agent with the error output, re-validate, then present.

## Challenge (between validate and present)

After the validator passes, run the adversarial challenge step per
`${CLAUDE_PLUGIN_ROOT}/reference/challenge-standards.md`: launch the **spec-challenger** agent with
the artifact + the same upstream inputs the architect had (never the architect's reasoning). It
files `reviews/CHALLENGE_<PHASE>.md` (feature mode: `features/<slug>/reviews/`) and never edits the
artifact. Same per-phase defaults as `/spec-kit:run`: **full loop** (route blockers/majors to the
architect to revise or rebut, then one scope-locked pass 2) for the product spec and technical
spec; **single pass** for the design spec and the plans; **skipped** on the light path or when the
user opts out. If the artifact changed, re-run the validator before presenting. The report is
presentation + audit trail only — never an input to downstream architects.

## Present, warn about staleness, stop

- **Present** the artifact the way `/spec-kit:run`'s corresponding checkpoint does: the parts that
  need human judgment first (requirement list and non-goals; architecture + constraint envelope;
  journeys; dependency order + tiers), with the validator result — and, when the challenge step
  ran, the disposition report, leading with anything `open — contested` (the human's ruling to
  make).
- **Warn about downstream staleness.** If artifacts downstream of the one just (re)generated already
  exist (e.g. a `build-plan.yaml` downstream of a regenerated `TECHNICAL_SPEC.md`), list them and note
  they may now be stale and should be regenerated or re-validated. Upstream edits do not auto-ripple.
- **Stop.** One phase per invocation is the point of these skills. Suggest the natural next step
  (the next phase's skill, or `/spec-kit:run` to resume the chain) — never launch it unasked.
