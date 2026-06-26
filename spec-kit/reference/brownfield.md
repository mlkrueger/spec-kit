# Brownfield: Adding a Feature to an Existing Repo (Reference)

> Standing methodology for operating `spec-kit` on a **pre-existing codebase** — adding a feature
> rather than building from an empty repo. Every architect agent and `/spec-kit:run` cite this when the
> work is feature-addition. The greenfield chain *invents* everything; brownfield must *respect what
> already exists* at every phase. When a project's own conventions conflict with these, the project
> wins — and in brownfield that is the whole point: the project almost always wins.

## The four principles

1. **Survey before specifying.** Never invent what already exists. Understand the repo — stack,
   architecture, conventions, test harness, existing features, and the existing ID/key namespace —
   before writing a single requirement. The survey is captured once in `REPO_MAP.md` (below) and
   threaded through every phase.
2. **Existing reality is a constraint.** The current stack, datastore, platform, auth model, and
   conventions are **de-facto hard constraints**. The technical spec *inherits* them rather than
   re-deciding them; diverging from one is an ADR with a rejected alternative, not a silent choice.
3. **Feature-scoped & collision-free.** A feature's artifacts live together and its IDs/keys never
   collide with prior features (see *Feature scoping*).
4. **Integrate & don't regress.** New work plugs into existing seams and **must not break existing
   behavior**. Existing tests and acceptance journeys stay green; integration points are explicit; the
   done-gate adds a regression requirement.

## `REPO_MAP.md` — the survey artifact (Phase 0)

Produced once at the start of a feature run and consumed by every downstream agent. It is the
brownfield analog of greenfield "intake." Structure:

- **Stack(s)** — each language/runtime + its test tooling, mapped to a `profiles/<stack>.md` key
  (`python`, `js-frontend`, `js-node`, …). The de-facto language/runtime constraints come from here.
- **Architecture & module seams** — the major components, where they live (real top-level paths), and
  the seams the new feature will attach to. Not exhaustive — the seams that matter for *this* feature.
- **Test harness** — how tests run (commands), the existing unit/integration setup and the **E2E
  harness** (so the acceptance plan reuses it, not rebuilds it), shared fixtures, and the **naming
  infixes already in use** (so new test files match).
- **Conventions** — code style, error-handling pattern, auth/authz model, observability, data-access
  pattern. These become inherited constraints.
- **Existing feature areas** — what is already built, so the new feature's boundary with existing
  behavior is explicit (what it integrates with, what it must not touch).
- **Existing ID/key namespace** — every `PR-*` already defined under `features/*/` and every ticket/key
  in prior plans, plus (if a tracker is connected) existing stamped keys. This is the collision set the
  new feature's IDs must avoid.
- **Inherited constraints** — the bullet list the technical spec will fold into `constraints.yaml` as
  hard constraints sourced from the existing codebase (owner: the codebase / the team).
- **Integration points & regression surface** — where the feature touches existing code, and which
  existing tests/journeys cover that area (the regression surface to keep green).

`REPO_MAP.md` is a review checkpoint: the user corrects the agents' understanding of their own repo
*once*, before it propagates through four phases.

## Feature scoping (layout + IDs)

- **Layout.** Each feature's artifacts live under **`features/<feature-slug>/`** — its own
  `PRODUCT_SPEC.md`, `TECHNICAL_SPEC.md`, `constraints.yaml`, `acceptance-plan.yaml`, `build-plan.yaml`.
  `REPO_MAP.md` may sit at repo root (shared survey) or in the feature dir; prefer the feature dir so a
  feature is self-contained. Multiple features coexist without clobbering.
- **Requirement IDs.** Prefix every `PR-*` with the feature slug: **`PR-<feature>-<req>`** (e.g.
  `PR-guest-checkout-place-order`). This guarantees no collision across features and is still valid
  kebab-case, so every schema and validator accepts it unchanged.
- **Ticket & milestone keys.** Prefix with the feature slug too (`<feature>-<ticket>`,
  milestone key `<feature>`), so build/acceptance plan keys — and the tracker stamps derived from them —
  never collide with another feature's.
- **Verify against the namespace.** Before minting any ID/key, check it against `REPO_MAP.md`'s existing
  ID/key namespace. A collision means either reuse the existing requirement (if it's the same thing) or
  pick a distinct slug.

No schema or validator changes are needed for any of this — feature-prefixed IDs and keys already
satisfy the existing kebab-case patterns. Brownfield is convention + methodology, not new machinery.

## How each phase changes

- **Product spec** — grounded by `REPO_MAP.md`'s existing feature areas. The new feature's scope is
  defined *relative to* what exists; non-goals call out existing behavior it must not change. IDs are
  feature-prefixed and checked against the namespace.
- **Technical spec** — folds `REPO_MAP.md`'s inherited constraints into `constraints.yaml` as hard
  constraints (rationale: "existing codebase"; owner: the team). Designs *to the existing seams*; any
  divergence from an existing convention is an explicit ADR. The architecture section names the
  integration points.
- **Acceptance plan** — **reuses the existing E2E harness** named in `REPO_MAP.md` (its
  `modulesInScope` reference the real existing harness/fixtures, not new ones). Adds new journeys for the
  feature, and notes which **existing journeys form the regression surface** that must stay green.
- **Build plan** — `modulesInScope` reference **real existing files to edit** as well as new files (the
  survey makes these verifiable). Tickets respect existing module seams. The `ci` done-gate ticket adds
  the regression requirement below.

## The regression gate

In brownfield, "done" requires not just the new tests green but the **existing suite still green**. The
build plan's always-emitted `ci` done-gate ticket gains a third clause:

- new build-plan unit/integration tests green, **and**
- the feature's acceptance journeys green, **and**
- **the pre-existing test suite (unit/integration + existing E2E journeys) still green** — no regression
  introduced by the feature.

Acceptance criteria on integration-touching build tickets should include "existing tests covering
<area> still pass."
