# Design-Phase Agent — Design

> A design (not a build) for one new agent that adds a **design phase** to the `spec-kit` chain: a
> **`design-spec-architect`** that turns an approved product spec into the visual + interaction
> contract every piece of frontend work obeys. It produces two human artifacts — a **Style Tile**
> (`STYLE_TILE.md`) and a **UI Style Guide** (`UI_STYLE_GUIDE.md`) — plus a machine-readable
> **`design-tokens.yaml`** with its own schema and `validate-design-tokens` tool. Written to mirror
> the conventions already established in `spec-kit` (agent body in `agents/*.md`, judgment-as-data in
> `reference/*-standards.md`, a machine-validatable side-artifact with a `validate-*` tool, the
> `PR-*` traceability spine, project-scoped agent memory, produce-then-stop hand-off).

---

## 1. Where it fits

`spec-kit` today has no notion of design. Engineering decisions (architecture, constraints) and test
decisions are first-class; the *visual and interaction language* a frontend is built in is nowhere —
so every frontend ticket re-invents color, type, spacing, and component behavior ad hoc, and there's
nothing for the build plan to point at. This agent fills that gap.

It sits **after the product spec** (it needs personas, the audience it's *not* for, and the UX
flows/states the product spec already describes) and **in parallel with the technical spec** — both
consume `PRODUCT_SPEC.md` and neither blocks the other. Its outputs are then consumed by the
**build-plan-architect** for any frontend ticket, exactly as `constraints.yaml` is consumed today.

```
                              ┌─▶ technical-spec-architect ─▶ TECHNICAL_SPEC.md + constraints.yaml ─┐
idea ─▶ product-spec-architect┤                                                                     ├─▶ build-plan-architect ─▶ build-plan.yaml
        ─▶ PRODUCT_SPEC.md     └─▶ design-spec-architect ────▶ STYLE_TILE.md                        │   (frontend tickets consume
                                  (frontend features only)     UI_STYLE_GUIDE.md + design-tokens.yaml┘    design-tokens.yaml +
                                                                                                          UI_STYLE_GUIDE.md)
                          acceptance-spec-architect ─▶ ACCEPTANCE_SPEC.md + acceptance-plan.yaml
```

The **artifact is the contract** between phases — exactly as `constraints.yaml` is the contract
between the technical spec and the build plan today. The design agent ends by handing off; it never
builds the page.

### Frontend-only, by design

This phase only runs when the feature has a user interface. For a headless service, a CLI, or a pure
API, it is **skipped** — the `run` skill makes it conditional (see §7), and the agent itself, if
invoked on a feature with no UX surface, says so and stops rather than inventing a visual language no
one will use.

### Design principles carried over from `spec-kit`

| Pattern in `spec-kit` | Applied to `design-spec-architect` |
|---|---|
| Agent persona + methodology in `agents/*.md`, rich `description` with `<example>` blocks | Same frontmatter shape and example-driven invocation |
| Domain judgment lives in `reference/*-standards.md` (standards-as-data) | `design-spec-standards.md` |
| Structured side-artifact + JSON Schema + `bin/validate-*` | `design-tokens.yaml` + `design-tokens.schema.json` + `validate-design-tokens` |
| `PR-*` traceability spine across phases | Tokens/components trace to the `PR-*` they serve; the style guide carries a coverage check |
| One sharp core discipline per agent | "Specify the language, not the layout" (below) |
| Project-scoped agent memory, durable/non-obvious only | Identical memory discipline |
| Produce the artifact, then **stop** and point to the next step | Hands off to the build plan; never writes the page |
| Brownfield: inherit existing conventions as constraints | Inherits an existing design system / token set as a hard constraint |

---

## 2. The agent — `design-spec-architect`

**Role.** A senior product designer / design-systems lead who turns an approved `PRODUCT_SPEC.md`
into the **visual and interaction language** the product is built in: the brand atmosphere, the
semantic token system, and the component contract. Not a layout designer drawing screens — a
*system* designer setting the rules that make every screen coherent.

**Core discipline (its "narrow waist" rule).**
> **Specify the language, not the layout.** Define the reusable system — tokens, type scale,
> component states, spacing rhythm — that *any* screen composes from. If a decision only makes sense
> for one specific screen, it belongs in that build ticket, not in the design system. Conversely,
> every visual choice a frontend ticket would otherwise improvise must be decided here, once.

That rule is what keeps this phase from sliding into per-screen comp work (which is the build plan's
job, traced to a `PR-*`) while still deciding everything a builder would otherwise guess at.

**Input.** Approved `PRODUCT_SPEC.md` (primary — for personas, audience, brand intent, UX states),
plus optionally an existing codebase / design system to respect (brownfield), and any brand inputs
the user supplies (logo, existing palette, brand guidelines, reference sites). Asks focused
clarifying questions when brand intent is thin — matching the other architects' "ask before
proceeding" behavior.

### Reference-controlled mode (extraction over invention)

The agent operates in one of two epistemic modes, and declares which up front:

- **Invention mode** — no design reference exists; the agent derives the language from the product
  spec's personas and brand intent. The original design above.
- **Reference-controlled mode** — a design reference exists, and **the reference controls**. The
  agent's job flips from *inventing* a language to *extracting* one: harvest the primitives and
  semantic roles from the source, fill only genuine gaps, and resolve every conflict in the
  reference's favor — even when the agent "has a better idea." The artifact outputs are identical
  (tile, guide, tokens); what changes is where their authority comes from.

**Reference precedence** (highest wins when sources conflict):

1. **A brand guide** the user supplies (PDF/markdown/URL) — the canonical statement of the brand.
2. **The design system already in the repo being extended** (tokens file, Tailwind config, CSS
   custom properties, component library) — brownfield reality, the de-facto system.
3. **Another repo** designated as the reference (e.g. "match our marketing site").
4. **Reference sites / screenshots / 'make it feel like X'** — aesthetic direction to emulate,
   loosest form.
5. **Invention** — fills whatever the references leave open, derived from the reference's own logic
   (e.g. extend the spacing scale by its existing ratio), never free-styled alongside it.

**Provenance.** In reference-controlled mode every extracted token carries a `source` field
(`design-tokens.yaml`) naming where it came from ("brand-guide §3.2", "acme-web/tailwind.config.js"),
and derived/invented gap-fills are marked as such — so a reviewer can see at a glance what is the
brand's voice and what is the agent's.

**The one legitimate deviation.** The accessibility floor is a hard constraint even on a brand
guide: when a reference pairing fails the WCAG AA contrast check, the agent must **flag it and
propose the nearest accessible variant** — never silently "fix" the brand, and never silently ship
the violation. The user decides; the validator enforces whatever ships. Everything else defers to
the reference.

**Style-tile behavior.** In reference-controlled mode the tile is a *distillation* of the reference
(proof the agent read it right — approved before systematization), not a tone proposal. The
approval question changes from "is this the right vibe?" to "did we extract your brand correctly?"

**Output.** `STYLE_TILE.md`, `UI_STYLE_GUIDE.md`, and `design-tokens.yaml` (validated).

**Suggested frontmatter** (mirrors the other architects):

```yaml
# agents/design-spec-architect.md
name: design-spec-architect
description: "Use this agent to turn an approved product spec into the visual + interaction contract
  for a frontend: a Style Tile (STYLE_TILE.md), a UI Style Guide (UI_STYLE_GUIDE.md), and a
  machine-readable design-tokens.yaml the build plan consumes. Frontend features only. <example>…</example>"
tools: Read, Write, Edit, Bash, Glob, Grep, WebFetch, WebSearch
model: opus
color: magenta
memory: project
```

---

## 3. Artifact 1 — `STYLE_TILE.md` (the atmosphere)

A **style tile** is the deliberately-early "is this the right *vibe*?" artifact — looser than a full
style guide, richer than a mood board. It exists to get directional sign-off on tone before any
systematic work hardens. Approve this *first*; the UI style guide and tokens are derived from an
approved tile.

Proposed structure (this is the set the user called out, made first-class):

- **Brand atmosphere & adjectives** — 3–5 words the UI must feel like ("calm, precise, editorial"),
  tied back to the product spec's personas and positioning. The north star every later choice serves.
- **Color palette with semantic names** — not raw hex grids but *roles*: `primary`, `accent`,
  `surface`, `surface-raised`, `text`, `text-muted`, `border`, `success`, `warning`, `danger`,
  `info`. Each role names its value(s) and where it's used. Light/dark intent stated here.
- **Typography stack** — heading vs. body families, the weights in use, the type scale (sizes +
  line-heights), and **example usage** (a real heading, a paragraph, a caption, a label rendered in
  prose so the relationships are visible).
- **Core UI elements** — buttons (primary/secondary/ghost/destructive), chips, inputs, cards, tags,
  pill nav — each *described* with its key states, so the look is unambiguous before the guide
  formalizes every state.
- **Example micro-layouts** — a hero fragment, a content block, a card grid: small compositions that
  prove the tokens and elements hang together, *as illustration* (clearly non-binding — real layouts
  are per-screen, traced to `PR-*` in the build plan).
- **Imagery & iconography treatment** — how photos/illustrations are framed, cropped, filtered, or
  textured; icon style (stroke weight, corner radius, fill vs. line); any motion atmosphere.
- **Anti-patterns** — what this brand explicitly is *not* (e.g. "no drop shadows," "never pure
  black," "no more than two type families"), so the system can't drift generic.

> **Rendering it.** Per the chosen approach, the agent writes the *spec*; it does **not** emit a
> runnable page. The build plan emits a ticket to build a non-shipping `/style-tile` reference page in
> the project's real stack (SvelteKit/Tailwind or whatever the repo uses), traced to a `PR-*`, that
> renders these tokens and elements as the canonical living reference. This keeps the agent
> stack-neutral and keeps the page TDD-traceable like any other build artifact.

---

## 4. Artifact 2 — `UI_STYLE_GUIDE.md` (the system)

Where the tile sets *tone*, the guide is the **contract a frontend ticket obeys**. It is the
systematic, exhaustive form: every token, every component state, the spacing and layout scale, and
the accessibility rules. Derived from an approved `STYLE_TILE.md`.

Proposed structure:

- **Design principles** — the tile's adjectives turned into actionable rules ("prefer weight over
  color for emphasis," "one accent per view").
- **Design tokens (canonical reference)** — the human-readable mirror of `design-tokens.yaml`: color,
  typography, spacing scale, radii, border, shadow/elevation, z-index, motion (durations, easings),
  breakpoints. Semantic tokens reference primitive tokens (e.g. `color.primary → palette.indigo.600`).
- **Spacing & layout system** — the base unit and scale, grid/columns, container widths, the
  breakpoint set, and the responsive rules.
- **Components** — for each component: anatomy, every **state** (default, hover, focus, active,
  disabled, loading, error, empty, selected), every **variant** (size/intent), the tokens it consumes,
  and usage do/don'ts. This is the bulk of the document and the part build tickets cite directly.
- **Accessibility rules** — contrast minimums (with the actual ratios the palette must meet), focus
  visibility, target sizes, motion-reduction, semantic/ARIA expectations. Stated as checkable rules,
  not aspirations — these become acceptance assertions on frontend tickets.
- **Content & microcopy tone** — voice for labels, errors, empty states (kept light; defers to a
  content guide if one exists).
- **Requirements traceability** — which `PR-*` each non-trivial component/decision serves; flags any
  UX state in the product spec with no component to render it (the design analog of the tech spec's
  matrix).

### `reference/design-spec-standards.md` (its standards-as-data)

The durable judgment the agent body points at (same split as the other agents):

- **The "specify the language, not the layout" discipline** and how to hold the line against
  per-screen comp work.
- **Semantic-token discipline** — the two-tier model (primitive → semantic), the required semantic
  role set, and the rule that components reference *semantic* tokens only (never raw primitives), so a
  re-theme is a token swap.
- **The accessibility bar** — WCAG AA as the default floor, the specific contrast/focus/target/motion
  rules, stated checkably.
- **The NFR/brand handoff** — how to read the product spec's user-facing tone ("feels calm,"
  "trustworthy") and turn it into concrete atmosphere + token decisions, the design analog of the tech
  spec's NFR→numeric handoff.
- **A failure-mode catalog to self-audit against**: generic AI-default aesthetic (the thing the
  `frontend-design` skills exist to fight); raw hex/px sprinkled instead of tokens; component states
  missing (no focus, no error, no empty, no loading); contrast that fails AA; a palette with no
  semantic mapping; tokens that trace to no requirement; per-screen layout masquerading as system;
  light/dark asymmetry; type scale with no rhythm.

---

## 5. The machine-readable artifact — `design-tokens.yaml`

The structural twist worth taking, mirroring `constraints.yaml` exactly: a downstream frontend build
ticket should consume tokens **programmatically** — generate the Tailwind config / CSS custom
properties from one source of truth, and lint that the built UI uses tokens rather than hardcoded
values. The guide's "Design tokens" section is the human mirror of this file.

Shape — a two-tier token model (primitives + semantic aliases), loosely aligned to the **W3C Design
Tokens (DTCG)** community format so it's portable into Style Dictionary / Tailwind generators:

```yaml
version: 1
spec: UI_STYLE_GUIDE.md
# Tier 1: primitives — the raw scale, no meaning attached.
primitives:
  color:
    indigo-600: "#4f46e5"
    slate-50:   "#f8fafc"
    slate-900:  "#0f172a"
  space:
    "1": "4px"
    "2": "8px"
    "4": "16px"
  font-size:
    sm: "14px"
    base: "16px"
    xl: "20px"
# Tier 2: semantic tokens — roles a component references. Each aliases a primitive.
tokens:
  - key: color-primary
    category: color
    value: "{primitives.color.indigo-600}"
    description: "Primary action / brand accent."
    tracesTo: [PR-onboarding-cta]        # optional: the requirement it serves
  - key: color-surface
    category: color
    value: "{primitives.color.slate-50}"
    description: "Default page/card background."
  - key: text-default
    category: color
    value: "{primitives.color.slate-900}"
    description: "Body text on surface."
    contrastsWith: color-surface          # drives the AA contrast check
    minContrast: 4.5
    source: "brand-guide §2.1"            # provenance (reference-controlled mode)
```

- **Schema** (`design-tokens.schema.json`, draft 2020-12): `key` unique/kebab-case; `category` ∈
  {color, typography, space, radius, border, shadow, motion, z-index, breakpoint}; required
  `value` + `description`; semantic `value`s must resolve to a declared primitive; `tracesTo` (when
  present) is a `PR-*`.
- **`bin/validate-design-tokens`**: schema check + the non-schema rules, mirroring
  `validate-constraints`:
  - key uniqueness and kebab-case;
  - every semantic `{primitives.…}` alias actually resolves to a declared primitive (no dangling
    references);
  - the **required semantic role set** is present (a UI without `color-primary`/`surface`/`text`/
    `danger`/… is incomplete);
  - every `contrastsWith` pair meets its `minContrast` (compute the WCAG ratio from the resolved
    hex — this makes the accessibility floor *mechanically enforced*, the highest-leverage check here);
  - every `tracesTo` id exists in the product spec (pass `--product-spec PRODUCT_SPEC.md`, same as
    `validate-constraints`).
- **Payoff**: a build ticket runs a generator over `design-tokens.yaml` to emit
  `tailwind.config`/CSS vars, and a lint rule rejects hardcoded color/spacing — the style guide stops
  being a PDF nobody follows and becomes a checked contract.

**Fallback** (the lighter path, if ever wanted): keep tokens as a structured section in
`UI_STYLE_GUIDE.md` and skip the artifact — the agent body simply omits the emit-and-validate step.
The chosen approach is the full artifact + validator.

---

## 6. How the build plan consumes the design phase

The seam that makes this phase pay off — additive, no breaking change to existing schemas:

- **`build-plan-architect`** gains `design-tokens.yaml` + `UI_STYLE_GUIDE.md` as **optional inputs**
  (present only for frontend features). When present:
  - It emits an early foundational ticket: **"generate the theme from `design-tokens.yaml`"** (Tailwind
    config / CSS vars) + the non-shipping `/style-tile` reference page, that later frontend tickets are
    `blockedBy`.
  - Each component/frontend ticket **references the relevant `UI_STYLE_GUIDE.md` component contract** in
    its description and adds token-conformance + the guide's accessibility rules to its
    `acceptanceCriteria` (e.g. "uses semantic tokens, no hardcoded hex; focus ring visible; meets AA").
  - A new optional ticket field `designRefs` (token keys / component names) can mirror `constraintRefs`,
    so a ticket declares which design contracts it honors — checkable later, but additive and not
    required for v1.
- **`technical-spec-architect`** stays in its lane: it still owns the *frontend architecture* (component
  framework, state management, rendering strategy) and records them as constraints, but **defers all
  visual specifics to the design phase** — the same membrane that keeps the product spec out of
  mechanism. A one-line note in `technical-spec-standards.md` records this boundary.

---

## 7. Threading into the `run` skill

The `run` skill's Phase 2 (technical spec) and a new design phase both consume the approved product
spec and don't depend on each other — so they run **in parallel**, with a single combined checkpoint,
*only when the feature has a UI*:

```
[1] product-spec-architect ─▶ PRODUCT_SPEC.md
        ⏸ approve product spec (and PR-* IDs)
[2] ┌─ technical-spec-architect ─▶ TECHNICAL_SPEC.md + constraints.yaml
    └─ design-spec-architect    ─▶ STYLE_TILE.md, UI_STYLE_GUIDE.md, design-tokens.yaml   (UI features only)
        ⏸ PRINCIPAL-ENG REVIEW (architecture + constraints)  +  DESIGN REVIEW (tile → guide → tokens)
[3] acceptance-spec-architect + build-plan-architect   (build plan now also consumes the design artifacts)
        ⏸ approve acceptance plan + build plan
```

- **Determining "has a UI":** the product spec's UX-flows/states section and personas make this
  obvious; the skill decides, and confirms with the user when ambiguous (same posture as greenfield vs.
  feature mode).
- **Design review checkpoint:** present the **tile first** for tone sign-off, then the guide + tokens,
  then the `validate-design-tokens` result. A failed validator blocks the checkpoint, same discipline
  as `validate-constraints`. Because the tile gates the guide, a tone change loops back cheaply before
  the systematic work hardens.
- **Coverage cross-check** (added to the existing one): every UX state in `PRODUCT_SPEC.md` has a
  component in the guide to render it; the AA contrast checks pass; frontend build tickets reference the
  guide.

---

## 8. Build order (when going from this design → files)

> **Status: BUILT** (including the reference-controlled mode above). All eight steps below are done;
> this document is retained as the design record.

1. Confirm placement + the full-artifact decision (done — both confirmed).
2. Write `reference/design-spec-standards.md` first — the durable judgment the agent body points at
   (same as how the other agents were built).
3. Write `reference/design-tokens-schema.md` (the human contract) + `reference/design-tokens.schema.json`
   (the machine schema), mirroring the constraints pair.
4. Write `bin/validate-design-tokens` (schema check + the non-schema rules above, especially the WCAG
   contrast computation), mirroring `bin/validate-constraints`.
5. Write `agents/design-spec-architect.md` (persona, methodology, the two-artifact output, the
   emit-and-validate step, hand-off, memory).
6. Thread the seams: add the optional design inputs + the theme/style-tile ticket guidance to
   `build-plan-architect.md`; add the "defer visual specifics" note to `technical-spec-standards.md`;
   add the conditional parallel design phase + design-review checkpoint to the `run` skill.
7. Add worked examples: `reference/examples/STYLE_TILE.example.md`, `UI_STYLE_GUIDE.example.md`,
   `design-tokens.example.yaml` — gold-standard targets, consistent with the existing example feature.
8. Update `OPEN_ITEMS.md` and the marketplace/plugin manifests if the agent list is enumerated there.

---

## 9. Open questions

- **Token format fidelity:** how strictly to follow the W3C DTCG `$value`/`$type` spelling vs. the
  lighter shape above. Lighter is easier to author and validate; strict DTCG is directly consumable by
  Style Dictionary. Leaning lighter for v1 with a note that it's DTCG-adjacent.
- **Where the `/style-tile` page lives** in a multi-feature (brownfield) repo — one shared reference
  page vs. per-feature. Probably one shared page that the design system as a whole owns; revisit when
  the first brownfield UI feature lands.
- **Relationship to the `frontend-design` / taste / soft skills** already installed — the agent should
  *reference* their aesthetic discipline (avoid generic AI defaults) in its standards rather than
  duplicate it. Worth a pointer, not a dependency.
