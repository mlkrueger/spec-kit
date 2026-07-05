# spec-kit

The **spec phase** of an AI SDLC / software factory, as one plugin. A chain of architect agents turns
an idea into the artifacts a build agent can execute against — each agent owns one phase, hands off a
machine-checkable artifact, and stops. Install once; invoke any agent on its own for incremental work,
or do a **full run** for greenfield.

## The chain

```
idea / brief / transcript
        │
        ▼
  product-spec-architect ──▶ PRODUCT_SPEC.md            (outside-in: what & why; defines PR-* IDs)
        │
        ├───────────────────────────────────────────────────────────────┐
        ▼                                                               ▼ (UI features only)
  technical-spec-architect ─▶ TECHNICAL_SPEC.md         design-spec-architect ─▶ STYLE_TILE.md
        │                  └─▶ constraints.yaml           (the visual language)   UI_STYLE_GUIDE.md
        │                      (hard/soft envelope)                               design-tokens.yaml
        │                                                               │
        ├─────────────────────────────────────────┬─────────────────────┘
        ▼                                           ▼
  acceptance-spec-architect ──▶ ACCEPTANCE_SPEC.md  build-plan-architect ──▶ BUILD_PLAN.md
        + acceptance-plan.yaml                            + build-plan.yaml
  (outer loop: E2E journeys, up-front)          (inner loop: test-first impl tickets, inline TDD;
                                                 frontend tickets consume the design contract)
```

Outside the chain, two standing agents work on what already exists: **test-audit-architect**
(evaluate the real suites, plan the backfill) and **ci-architect** (design or audit the pipeline).

Everything is linked by a **`PR-*` traceability spine** (`reference/traceability.md`): idea →
requirement → design decision → ticket, so "is every requirement designed, built, and tested?" is a
lookup, not a judgment call.

## What's here today

| Component | What it does |
|---|---|
| `agents/product-spec-architect.md` | Turns an idea/brief/transcript into `PRODUCT_SPEC.md` — requirements as observable behaviors, each with a `PR-*` ID. Core rule: *describe behavior, never mechanism.* |
| `agents/technical-spec-architect.md` | Turns `PRODUCT_SPEC.md` into `TECHNICAL_SPEC.md` + a validated `constraints.yaml`. Core rule: *decide the load-bearing things; defer the rest.* |
| `agents/build-plan-architect.md` | Decomposes `TECHNICAL_SPEC.md` (+ `constraints.yaml` + product spec) into a dependency-ordered, test-first `build-plan.yaml` + `BUILD_PLAN.md`. Core rule: *every ticket is independently buildable and test-first.* |
| `agents/acceptance-spec-architect.md` | Turns the product spec's acceptance criteria into up-front E2E journeys (`acceptance-plan.yaml` + `ACCEPTANCE_SPEC.md`), the outer TDD loop. Core rule: *test the journey, not the click.* |
| `agents/design-spec-architect.md` | Turns the product spec into the visual + interaction contract (UI features only): `STYLE_TILE.md` (tone, approved first) + `UI_STYLE_GUIDE.md` (the contract frontend tickets cite) + validated `design-tokens.yaml`. **Reference-controlled mode** when a brand guide / existing design system / reference repo or site exists: extraction with `source` provenance, the reference wins conflicts, AA violations flagged never silently fixed. Core rule: *specify the language, not the layout.* |
| `reference/design-spec-standards.md` | The design brain: the two modes + reference precedence, extraction discipline, semantic-token model + required role floor, the checkable accessibility bar, the brand handoff, failure-mode catalog. |
| `reference/design-tokens-schema.md` + `.schema.json` | The token contract (prose + JSON Schema): two tiers, `contrastsWith`/`minContrast`, `source` provenance, `tracesTo`. |
| `bin/validate-design-tokens` | Validates `design-tokens.yaml`: schema + alias resolution, the required role floor, **computed WCAG contrast on declared pairs**, `tracesTo` existence, reference-mode provenance warnings. |
| `skills/design-spec/` | `/spec-kit:design-spec` — one-off design phase with the tile-first checkpoint. |
| `reference/product-spec-standards.md` | Outside-in discipline, banned-vocabulary filter, INVEST bar, the NFR handoff, failure-mode catalog. |
| `reference/technical-spec-standards.md` | Hard/soft constraint taxonomy, ADR-lite, cross-cutting checklist, failure-mode catalog. |
| `reference/testing-standards-shared.md` | Cross-layer testing discipline: ticket specificity, determinism, "a spec precedes the suite." Shared by both downstream test agents. |
| `reference/unit-integration-standards.md` | Lower-layer brain referenced inline by `build-plan-architect`: lowest-layer-that-proves-it, clock injection, the failure-mode catalog, red→green→refactor DoD. |
| `reference/acceptance-standards.md` | E2E-only brain for `acceptance-spec-architect`: journey-not-click, mocked-backend harness, separate CI job. |
| `agents/test-audit-architect.md` | Audits an **existing** repo's suites — scopable to unit / integration / e2e / all — against the testing standards: inventories + runs the real suites, scores each layer, emits evidence-grounded `TEST_AUDIT.md` + (on request) a validated test-backfill `build-plan.yaml`. Core rule: *every finding cites evidence; every gap names what breaks unobserved.* |
| `reference/test-audit-standards.md` | The audit brain: evidence to collect, per-layer scoring dimensions, finding format, and the backfill-plan conventions (characterization discipline, refactor-blocks-test ordering). |
| `skills/test-audit/` | `/spec-kit:test-audit [unit\|integration\|e2e\|all]` — one-off audit wrapper around the agent. |
| `agents/ci-architect.md` | The pipeline as its own first-class concern, dual mode. **Design:** `CI_SPEC.md` — gates traced to spec'd bars, budgets, required checks, environments/promotion, secrets, rollback — which the build plan turns into `ci` tickets. **Audit:** `CI_AUDIT.md` — the bar-diff (spec'd vs. wired-and-blocking) + findings, optional remediation `build-plan.yaml`. Core rule: *every gate traces to a bar; every bar is wired to a gate.* |
| `reference/ci-standards.md` | The CI/CD brain: the two-job shape, required-checks enforcement, flake policy, speed budgets, build-once promotion, secrets, deploy/rollback, forge notes, the audit dimensions + failure-mode catalog. |
| `skills/ci/` | `/spec-kit:ci [design\|audit]` — one-off pipeline design or audit wrapper around the agent. |
| `reference/traceability.md` | The shared `PR-*` requirement-ID spine. |
| `reference/constraints-schema.md` + `.schema.json` | The constraint-envelope contract (prose + JSON Schema), plus the design note for future constraint linting. |
| `reference/build-plan-schema.md` + `.schema.json` | The build-plan contract (prose + JSON Schema): inline `tddCases`, `constraintRefs`, `tracesTo`, `tier`. |
| `reference/acceptance-plan-schema.md` + `.schema.json` | The acceptance-plan contract (prose + JSON Schema): `e2e`/`harness` layers, Given/When/Then `cases`, `tracesTo`. |
| `bin/validate-constraints` | Validates `constraints.yaml` against the schema **and** non-schema rules (key uniqueness, escape-hatch rules, `tracesTo` existence against a required product spec). |
| `bin/validate-build-plan` | Validates `build-plan.yaml`: schema + key uniqueness, referential integrity, acyclicity, real paths, **`constraintRefs` exist** in the envelope, **`tracesTo` exist** in the product spec. |
| `bin/validate-acceptance-plan` | Validates `acceptance-plan.yaml`: schema + the four structural checks + **`tracesTo` exist** in the product spec. |
| `skills/run/SKILL.md` | `/spec-kit:run` — drives the full four-agent chain with a human-approval checkpoint between every phase (incl. the principal-eng review before the build plan). |
| `skills/survey/`, `skills/product-spec/`, `skills/technical-spec/`, `skills/acceptance-plan/`, `skills/build-plan/` | Per-phase skills: run any single phase as a one-off — resolve inputs, invoke the architect, validate, present, stop. Shared ceremony in `reference/single-phase.md`. |
| `reference/single-phase.md` | The contract every per-phase skill obeys: input resolution, mode detection, validate-before-present, staleness warnings, present-and-stop. |
| `skills/publish-linear/` + `skills/publish-jira/` | Publish a neutral `build-plan.yaml` / `acceptance-plan.yaml` to Linear or Jira, idempotently via key stamping. |
| `reference/publishing.md` | The shared publishing contract both publishers obey: plan-kind detection, body rendering, idempotency, config. |
| `reference/profiles/*.md` | Per-stack cards (`python`, `js-frontend`, `js-node`, `go`) — runner, single-test command, test-file naming infix, mocking seam, gotchas. Consumed by the build/acceptance agents. |
| `reference/brownfield.md` | Brownfield methodology: survey-before-specifying (tiered: scoped vs. full), existing reality as constraints, repo grounding, feature scoping, **change mode** (behavior deltas, `Modifies:` links, test migration), the regression gate, and the light path for small changes. Cited by all agents + `/spec-kit:run`. |
| `reference/examples/` | A worked guest-checkout set sharing one `PR-*` spine: `REPO_MAP`, `PRODUCT_SPEC`, `TECHNICAL_SPEC`, `constraints`, `build-plan`, `acceptance-plan`, `STYLE_TILE`, `UI_STYLE_GUIDE`, and `design-tokens` examples. |

## Full run vs. single agent

- **Greenfield:** `/spec-kit:run <brief>` drives the whole chain with an approval checkpoint between
  every phase. You review and approve each artifact before the next phase consumes it; the principal-eng
  review lands between the technical spec and the build plan.
- **Brownfield:** `/spec-kit:run` on a populated repo first runs a **Phase-0 survey** — scoped to the
  feature by default, full-repo when it's the first run or the blast radius is wide — which you review
  before it propagates. Then the chain runs in *feature mode*: artifacts under `features/<slug>/`,
  feature-prefixed IDs (`PR-<slug>-<req>`), existing conventions inherited as hard constraints, the
  existing E2E harness reused, and a **regression clause** on the done-gate so the pre-existing suite
  stays green. Requirements that *modify* existing behavior get **change mode**: a before/after
  behavior delta, `Modifies:` supersession links, and explicit test-migration tickets instead of a
  blanket "existing tests stay green." Small single-seam changes can take the **light path** (scoped
  survey → mini spec → build plan). See `reference/brownfield.md`.
- **Incremental:** run any phase as a one-off via its skill — `/spec-kit:survey`,
  `/spec-kit:product-spec`, `/spec-kit:technical-spec`, `/spec-kit:acceptance-plan`,
  `/spec-kit:build-plan`. Each resolves its upstream inputs (argument > `features/<slug>/` > cwd),
  invokes the architect, validates, presents the same review its `/spec-kit:run` checkpoint would,
  warns when downstream artifacts go stale, and stops. The plans validate independently, so you only
  redo what changed.

## Publishing

The neutral plans publish to a tracker via a publisher skill that maps intent → tracker using per-team
config in `${CLAUDE_PROJECT_DIR}/.spec-kit/publisher.yaml`:

- `/spec-kit:publish-linear` and `/spec-kit:publish-jira` — both obey one shared contract
  (`reference/publishing.md`); they differ only in tracker mapping. Idempotent via `key` stamping
  (a `skp:<key>` label + a `<!-- skp-key: <key> -->` body marker), so re-publishing updates instead of
  duplicating. Default is skip-if-exists; `--update` overwrites managed fields; `--dry-run` previews.

## Install

```
/plugin marketplace add mkrueger/agent-specs
/plugin install spec-kit@agent-specs
```

## Legacy

`spec-kit` is the successor to the standalone `test-spec` plugin, now fully folded in: its testing
judgment became `testing-standards-shared.md` + `unit-integration-standards.md` +
`acceptance-standards.md`, its profiles migrated, and its `publish-linear` generalized. The old
standalone **ticket-plan** output is retired — its job is split between the build plan's inline
`tddCases` and the acceptance plan — but `ticket-plan.schema.json` + `bin/validate-ticket-plan` are
retained so a **pre-existing** `ticket-plan.yaml` can still be published. No agent produces one; for new
work use the build and acceptance plans.

## Design principles (carried from `test-spec`)

- **The artifact is the contract** between phases. Each agent produces a validatable artifact, then
  stops — it never does the next phase's job.
- **Judgment lives in `reference/*-standards.md`** as standards-as-data; agent bodies point at them.
- **Machine-checkable side-artifacts** (`constraints.yaml`, and the coming `build-plan.yaml` /
  `acceptance-plan.yaml`) each get a `bin/validate-*` tool.
- **Project-scoped agent memory**, committed to the repo, durable/non-obvious only.

## Usage

1. Run `product-spec-architect` on your idea/brief/transcript → `PRODUCT_SPEC.md`.
2. Run `technical-spec-architect` on the approved product spec → `TECHNICAL_SPEC.md` +
   `constraints.yaml` (auto-validated).
3. Validate constraints manually anytime (a product spec is required):
   ```sh
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-constraints constraints.yaml            # resolves a sibling PRODUCT_SPEC.md
   ${CLAUDE_PLUGIN_ROOT}/bin/validate-constraints constraints.yaml --product-spec path/to/PRODUCT_SPEC.md
   ```
