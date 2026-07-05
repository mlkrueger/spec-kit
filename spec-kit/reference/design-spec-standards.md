# Design Spec Standards (Reference)

> Reference document for the **design-spec-architect** agent. These are the standing standards for
> what a good design specification looks like — the visual + interaction contract a frontend is
> built in — independent of any one product or aesthetic. The design spec describes the **language**
> (tokens, type, spacing, component states) that every screen composes from; per-screen layout is
> the build plan's job. When a project's own conventions conflict with these, the project wins, but
> call out the divergence.

## The one discipline everything else serves

**Specify the language, not the layout.** Define the reusable system — tokens, type scale, component
states, spacing rhythm — that *any* screen composes from. If a decision only makes sense for one
specific screen, it belongs in that build ticket, not in the design system. Conversely, every visual
choice a frontend ticket would otherwise improvise must be decided here, once. The membrane runs both
ways: no screen comps in the system, no improvised hex codes in the screens.

## The two modes: invention vs. reference-controlled

Declare the mode before designing anything; it decides where authority comes from.

- **Invention mode** — no design reference exists. Derive the language from the product spec's
  personas, positioning, and brand intent; ask focused clarifying questions when intent is thin.
- **Reference-controlled mode** — a design reference exists, and **the reference controls**. The job
  flips from inventing to **extracting**: harvest the primitives and semantic roles from the source,
  fill only genuine gaps, and resolve every conflict in the reference's favor — even over a "better
  idea." Same three artifacts; different epistemics.

**Reference precedence** (highest wins when sources conflict):

1. **A brand guide** the user supplies (PDF / markdown / URL) — the canonical statement of the brand.
2. **The design system already in the repo** being extended (tokens file, Tailwind config, CSS custom
   properties, component library) — brownfield reality, the de-facto system.
3. **Another repo designated as reference** ("match our marketing site").
4. **Reference sites / screenshots / "feel like X"** — aesthetic direction, loosest form.
5. **Invention** — fills what the references leave open, derived from the reference's own logic
   (extend the spacing scale by its existing ratio; darken the brand hue for a hover state), never
   free-styled alongside it.

**Extraction discipline:**

- **Provenance on every extracted token.** The `source` field names where it came from
  ("brand-guide §3.2", "acme-web/tailwind.config.js:14"); gap-fills are marked
  ("derived: brand ratio", "invented: no reference"). A reviewer must be able to see at a glance
  what is the brand's voice and what is the agent's.
- **No cherry-picking.** Extract the reference's system, including the parts you'd have chosen
  differently. Deviating from the reference is the *user's* decision, surfaced explicitly — never a
  silent improvement.
- **Ground extraction like brownfield code** (brownfield principle 5): read the actual
  config/tokens/CSS when the reference is a repo; cite real files. A "reference-controlled" system
  whose tokens trace to nothing extracted is invention wearing a costume.

**The one legitimate deviation: the accessibility floor.** When a reference pairing fails the AA
contrast bar, **flag it and propose the nearest accessible variant** — never silently "fix" the
brand, never silently ship the violation. The user decides; `validate-design-tokens` enforces
whatever ships.

## Semantic-token discipline (the two-tier model)

- **Tier 1 — primitives**: the raw scales (`indigo-600`, `space.4`, `font-size.xl`), no meaning
  attached.
- **Tier 2 — semantic tokens**: roles (`color-primary`, `text-muted`), each aliasing a primitive.
- **Components reference semantic tokens only** — never primitives, never raw hex/px. That is what
  makes a re-theme a token swap and what the token-conformance lint on frontend tickets enforces.
- **The required role floor** (a UI without these is incomplete; `validate-design-tokens` checks
  them): `color-primary`, `color-surface`, `text-default`, `text-muted`, `color-border`,
  `color-danger`, `color-success`. Add roles beyond the floor as the product needs them
  (`surface-raised`, `color-warning`, `color-info`, `focus-ring`, …) — but never skip the floor.
- **Light/dark intent is stated on the role**, not improvised per screen; asymmetric themes (a dark
  mode with missing roles) are a failure mode.

## The accessibility bar (checkable, not aspirational)

WCAG 2.2 AA is the default floor, stated as rules a test can check:

- **Contrast** — text vs. its surface ≥ 4.5:1 (normal) / 3:1 (large text and essential UI
  boundaries). Declare the load-bearing pairs on the tokens (`contrastsWith` + `minContrast`) so the
  validator computes real ratios from resolved hex — the floor is *mechanically enforced*.
- **Focus** — a visible focus indicator on every interactive element; the focus style is a token +
  component-state decision made here, not per screen.
- **Targets** — minimum interactive target size (24×24 CSS px floor; 44×44 preferred on touch).
- **Motion** — durations/easings are tokens; every non-essential animation respects
  `prefers-reduced-motion`.
- **Semantics** — color is never the only channel for a state (pair with weight/icon/text).

These become acceptance assertions on frontend build tickets, not decoration in a document.

## The brand handoff (the design analog of the NFR handoff)

The product spec states brand intent as the user feels it ("calm," "trustworthy," "playful") —
personas and positioning, never palettes. The design spec translates each adjective into concrete,
attributable decisions (type scale rhythm, palette temperature, radius, motion character), the same
way the technical spec turns "feels instant" into a p99. An adjective with no consequent decision is
decoration; a visual decision serving no adjective (or requirement) is taste with no anchor —
justify or cut either.

## Tile before guide

The style tile (atmosphere: adjectives, palette roles, type, core elements, anti-patterns) is
approved **before** the systematic work hardens — tone changes loop cheaply at the tile,
expensively at the guide. In reference-controlled mode the tile is a *distillation* of the reference
(the approval question becomes "did we extract your brand correctly?"), not a tone proposal. The
guide and tokens are derived from an approved tile, never in parallel with it.

## Failure-mode catalog (audit every design spec against this)

- **Generic AI-default aesthetic** — the interchangeable purple-gradient look; nothing traces to
  this brand's adjectives or reference. The reference/brand handoff exists to prevent exactly this.
- **Invention leaking into a referenced system** — tokens that contradict or "improve on" the
  controlling reference without a flagged decision.
- **Missing provenance** — reference-controlled mode with `source`-less tokens.
- **Raw values sprinkled** — hex/px in component specs instead of semantic tokens.
- **Missing states** — components specified without focus, error, empty, loading, disabled.
- **Contrast failures** — palette pairs that fail AA; undeclared `contrastsWith` on load-bearing
  pairs (an unchecked pair is an unenforced floor).
- **Palette with no semantic mapping** — pretty primitives, no roles; the required floor absent.
- **Tokens tracing to no requirement** — a token set serving no `PR-*` or UX state; justify (core
  role floor) or cut.
- **Per-screen layout masquerading as system** — screen comps in the guide.
- **Light/dark asymmetry** — a second theme with missing or unconsidered roles.
- **Type scale with no rhythm** — sizes that follow no ratio and line-heights that follow no grid.

## Output discipline

- Three artifacts, in authority order: `STYLE_TILE.md` (approved first), `UI_STYLE_GUIDE.md` (the
  contract build tickets cite), `design-tokens.yaml` (validated; the machine mirror of the guide's
  token section).
- The agent writes the *spec*; it never emits a runnable page. The living reference page
  (`/style-tile`) is a build-plan ticket in the project's real stack, traced like any other.
- **No UX surface, no design phase.** Invoked on a feature with no UI, say so and stop — never
  invent a visual language nothing will render.
