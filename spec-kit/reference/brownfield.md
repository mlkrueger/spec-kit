# Brownfield: Working in an Existing Repo (Reference)

> Standing methodology for operating `spec-kit` on a **pre-existing codebase** — rather than building
> from an empty repo. Every architect agent and `/spec-kit:run` cite this when the work touches an
> existing codebase. The greenfield chain *invents* everything; brownfield must *respect what already
> exists* at every phase. When a project's own conventions conflict with these, the project wins —
> and in brownfield that is the whole point: the project almost always wins.

## The two shapes of brownfield work

- **Feature-addition** — adding behavior the repo doesn't have. Existing behavior is context and
  constraint; nothing that works today is supposed to work differently tomorrow.
- **Behavior-change** — modifying, replacing, or removing behavior the repo already has (a redesign,
  a rule change, a refactor with observable effects, a deprecation). Existing behavior is the
  *subject* of the work, not just its surroundings.

Most real work mixes both: a feature that also changes one existing flow. Apply the change-mode
discipline (below) to the changed parts and the addition discipline to the rest — the split is
per-requirement, not per-run.

## The five principles

1. **Survey before specifying.** Never invent what already exists. Understand the repo — stack,
   architecture, conventions, test harness, existing features, and the existing ID/key namespace —
   before writing a single requirement. The survey is captured in a `REPO_MAP.md` (full or scoped —
   see *The survey is tiered*) and threaded through every phase.
2. **Existing reality is a constraint.** The current stack, datastore, platform, auth model, and
   conventions are **de-facto hard constraints**. The technical spec *inherits* them rather than
   re-deciding them; diverging from one is an ADR with a rejected alternative, not a silent choice.
3. **Feature-scoped & collision-free.** A feature's artifacts live together and its IDs/keys never
   collide with prior features (see *Feature scoping*).
4. **Integrate, and change deliberately.** New work plugs into existing seams. Behavior that is not
   the subject of the work **must not change** — its tests stay green unchanged. Behavior that *is*
   the subject of the work changes **on purpose**: the old behavior is documented, the new behavior
   specified, and the tests covering it *migrated*, never deleted-and-forgotten (see *Change mode*).
5. **Ground in the repo, not the survey.** `REPO_MAP.md` is a map, not the territory — it goes stale
   and it summarizes. Before emitting a real path, seam, interface, or convention claim in any
   artifact, **verify it against the actual repo** (Glob/Grep/Read the real files) — never emit a
   path or convention you haven't seen in this run. Where an artifact leans on a verified fact, say
   what was checked (e.g. "verified: `src/checkout/` seam, `*_test.py` infix"). Outputs that read
   greenfield-ish — invented paths, generic conventions, ignored seams — are the primary brownfield
   failure mode; this principle is the antidote.

   **Verify cheaply — grounding is not re-surveying.** Match the verification to the claim:
   - *Existence claims* (a path in `modulesInScope` is real): verify **in bulk** — one Glob per
     directory or a single `ls`/`test -f` Bash loop over every candidate path, not one Read per
     file. Existence never requires reading file contents.
   - *Content claims* (an interface signature, a seam, a naming convention the artifact states):
     Read only the specific files the claim is about, and prefer Grep or a targeted Read
     (offset/limit) over pulling in whole files.
   - *Everything else*: trust the survey. `REPO_MAP.md` exists so downstream phases don't re-explore
     the repo — re-reading source the survey already mapped, "to be safe," is the context-bloat
     failure mode that mirrors greenfield-ish output on the cost side. Re-open an area only when
     grounding contradicts the map (then also refresh the map section).

## The survey is tiered

A full repo survey before a two-file feature is ceremony that kills adoption; no survey at all
produces greenfield-ish output. Two tiers:

- **Scoped survey (the default).** Survey only what *this* feature touches: the seams it attaches
  to, the conventions of the modules it will edit, the tests covering the area (the regression
  surface), and the ID/key namespace. Same section structure as the full map, feature-scoped depth,
  written to `features/<slug>/REPO_MAP.md`. State explicitly what was **not** surveyed so no one
  mistakes it for the full map. Minutes, not hours.
- **Full survey.** The complete map — every stack, all major seams, the whole harness and namespace —
  written to a repo-root `REPO_MAP.md`. Warranted on the **first** spec-kit run in a repo, and for
  wide-blast-radius work (cross-cutting changes, migrations, anything touching >2–3 subsystems).
  It is a shared, durable artifact: later features **delta off it** (scoped survey = read the root
  map + verify/extend the sections this feature touches) instead of re-surveying from scratch.
  Refresh a section when grounding (principle 5) contradicts it — a stale map that loses to reality
  is working as intended; silently trusting it is not.

Either tier is a review checkpoint: the user corrects the agents' understanding of their own repo
*once*, before it propagates through the phases.

### `REPO_MAP.md` structure (both tiers)

- **Stack(s)** — each language/runtime + its test tooling, mapped to a `profiles/<stack>.md` key
  (`python`, `js-frontend`, `js-node`, …). The de-facto language/runtime constraints come from here.
- **Architecture & module seams** — the major components, where they live (real top-level paths), and
  the seams the work attaches to. Scoped tier: only the seams that matter for *this* feature.
- **Test harness** — how tests run (commands), the existing unit/integration setup and the **E2E
  harness** (so the acceptance plan reuses it, not rebuilds it), shared fixtures, and the **naming
  infixes already in use** (so new test files match).
- **Conventions** — code style, error-handling pattern, auth/authz model, observability, data-access
  pattern. These become inherited constraints.
- **Existing feature areas** — what is already built, so the boundary with existing behavior is
  explicit (what the work integrates with, what it must not touch, and — in change mode — what it
  deliberately changes).
- **Existing ID/key namespace** — every `PR-*` already defined under `features/*/` and every
  ticket/key in prior plans, plus (if a tracker is connected) existing stamped keys. This is the
  collision set the new feature's IDs must avoid.
- **Inherited constraints** — the bullet list the technical spec will fold into `constraints.yaml` as
  hard constraints sourced from the existing codebase (owner: the codebase / the team).
- **Integration points & regression surface** — where the work touches existing code, and which
  existing tests/journeys cover that area. In change mode, split the surface (see below).
- **(Scoped tier only) Not surveyed** — the areas deliberately left unmapped.

## Feature scoping (layout + IDs)

- **Layout.** Each feature's artifacts live under **`features/<feature-slug>/`** — its own
  `PRODUCT_SPEC.md`, `TECHNICAL_SPEC.md`, `constraints.yaml`, `acceptance-plan.yaml`, `build-plan.yaml`,
  and (scoped-tier) `REPO_MAP.md`. The full-tier `REPO_MAP.md` sits at repo root (shared survey).
  Multiple features coexist without clobbering.
- **Requirement IDs.** Prefix every `PR-*` with the feature slug: **`PR-<feature>-<req>`** (e.g.
  `PR-guest-checkout-place-order`). This guarantees no collision across features and is still valid
  kebab-case, so every schema and validator accepts it unchanged.
- **Ticket & milestone keys.** Prefix with the feature slug too (`<feature>-<ticket>`,
  milestone key `<feature>`), so build/acceptance plan keys — and the tracker stamps derived from
  them — never collide with another feature's.
- **Verify against the namespace.** Before minting any ID/key, check it against the existing ID/key
  namespace in the survey. A collision means either reuse the existing requirement (if it's the same
  thing) or pick a distinct slug.

No schema or validator changes are needed for any of this — feature-prefixed IDs and keys already
satisfy the existing kebab-case patterns. Brownfield is convention + methodology, not new machinery.

## Change mode (behavior-change work)

When a requirement modifies existing behavior, four things change:

- **The behavior delta.** The product spec documents, per changed requirement, the **current
  behavior** (as observed — describing what the system does today is observation, not solutioning)
  and the **new behavior**, side by side. A change spec without the "before" column is an addition
  spec in disguise — reviewers can't see what users will lose or notice.
- **`Modifies:` links.** A requirement that changes existing behavior carries a **`Modifies:`**
  annotation naming what it supersedes: the existing `PR-*` if the behavior was specced by a prior
  feature, otherwise a pointer to the delta row describing the un-specced current behavior. Per the
  traceability discipline, a superseded `PR-*` is **retired, not edited** — note the supersession in
  the old spec if it's in this repo. New IDs are minted normally (`PR-<feature>-<req>`).
- **The regression surface splits.** Existing tests covering the touched area divide into:
  - **must-stay-green unchanged** — tests covering behavior that is *not* the subject of the change;
  - **must-be-migrated** — tests asserting the *old* behavior, which are now *expected to fail* once
    the change lands. Each is updated to assert the new behavior (or explicitly retired with the
    requirement it covered), via dedicated migration tickets — never bulk-edited to pass, never
    silently deleted.
  The survey names the split; the plans act on it. A change plan whose existing tests "all still
  pass" untouched didn't change anything — or changed it without the tests noticing, which is worse.
- **Superseded journeys.** Existing acceptance journeys asserting the old behavior are named in
  `ACCEPTANCE_SPEC.md` as superseded and get updated journeys tracing to the new `PR-*`; journeys
  outside the change stay in the must-stay-green set.

## How each phase changes

- **Product spec** — grounded by the survey's existing feature areas. Additions: scope defined
  *relative to* what exists; non-goals call out existing behavior the work must not change. Changes:
  the behavior delta + `Modifies:` links (above). IDs are feature-prefixed and checked against the
  namespace.
- **Technical spec** — folds the survey's inherited constraints into `constraints.yaml` as hard
  constraints (rationale: "existing codebase"; owner: the team). Designs *to the existing seams*,
  verified in the repo, not just the map; any divergence from an existing convention is an explicit
  ADR. In change mode, addresses migration/compatibility explicitly (data migration, API versioning,
  feature-flag or cutover strategy, rollback) — a behavior change with no rollout story is unfinished.
- **Acceptance plan** — **reuses the existing E2E harness** named in the survey (its
  `modulesInScope` reference the real existing harness/fixtures, verified, not new ones). Adds new
  journeys, names the **must-stay-green** existing journeys, and in change mode the **superseded**
  journeys + their replacements.
- **Build plan** — `modulesInScope` reference **real existing files to edit**, verified in the repo
  before emitting. Tickets respect existing module seams. In change mode, emits explicit
  **test-migration tickets** for the must-be-migrated set (update the assertion to the new behavior,
  traced to the new `PR-*`), ordered `blockedBy` the behavior change they follow. The `ci` done-gate
  ticket adds the regression clause below.

## The regression gate

In brownfield, "done" requires not just the new tests green but the existing suite accounted for. The
build plan's always-emitted `ci` done-gate ticket gains:

- new build-plan unit/integration tests green, **and**
- the feature's acceptance journeys green, **and**
- the **must-stay-green** set (pre-existing unit/integration tests + existing E2E journeys outside
  the change) still green **unchanged**, **and**
- *(change mode)* the **must-be-migrated** set fully migrated — every test that asserted the old
  behavior now asserts the new one or is explicitly retired alongside its requirement; none skipped,
  none deleted without a record.

Acceptance criteria on integration-touching build tickets should include "existing tests covering
<area> still pass"; on behavior-changing tickets, "tests formerly asserting <old behavior> are
migrated per the delta."

## The light path (small changes)

The full four-phase chain is the wrong tool for a bugfix-sized, single-seam change. The light path
keeps the spine and the gates while dropping the ceremony:

1. **Scoped survey, inline.** Ground the seam, conventions, tests, and namespace for the touched
   area (minutes; a `REPO_MAP.md` file is optional — grounding is not).
2. **Mini product spec.** A short `PRODUCT_SPEC.md` — the problem, one to three `PR-*` requirements
   as observable behaviors (with the behavior delta + `Modifies:` if changing behavior), acceptance
   criteria. No personas/metrics ceremony. This keeps `tracesTo` validatable.
3. **Skip the technical spec** when the change makes no architectural decision — the existing repo
   *is* the constraint envelope. (Tickets then simply carry no `constraintRefs`; the build-plan
   validator only requires a constraints file when tickets reference one.) If you catch yourself
   making a load-bearing decision — new dependency, new seam, a convention divergence — that's the
   signal you've left the light path; do a real technical spec.
4. **Build plan** as normal (it can be a handful of tickets), including the regression clause and
   any test-migration tickets. Acceptance plan only if the change alters a user-observable journey.

The light path is still test-first and still traced — what it removes is survey breadth and
spec ceremony, not the discipline.
