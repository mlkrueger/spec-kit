# UI Style Guide — Acme Storefront (guest checkout)

> Example UI style guide for the guest-checkout example feature, produced by
> **design-spec-architect** from the approved `STYLE_TILE.example.md`. This is the **contract every
> frontend build ticket obeys**; `design-tokens.example.yaml` is its machine mirror.
> **Mode: reference-controlled** (acme-brand-guide.pdf + storefront/tailwind.config.js) — the
> reference wins conflicts; provenance is on every token.

## Design principles

1. **One accent per view.** `color-primary` appears on exactly one primary action; everything else
   earns emphasis with weight, not color. *(tile: "assured")*
2. **Prefer weight over color** for hierarchy — 600 vs. 400 before another hue.
3. **Nothing floats.** Hairline borders, no shadows/elevation. *(brand guide §4)*
4. **Semantic tokens only.** Components reference roles (`color-primary`), never primitives or raw
   hex/px — a re-theme must be a token swap.

## Design tokens (canonical reference)

The full set lives in `design-tokens.example.yaml` (validated: role floor, alias resolution, AA
contrast on load-bearing pairs). Summary: color roles per the tile; type scale 14–24 on Inter
400/600; spacing on a 4px base (`space-stack` 16px, `space-inline` 8px); `radius-control` 4px /
`radius-card` 8px; `motion-interaction` 120ms.

## Spacing & layout

- 4px base unit; vertical rhythm between blocks is `space-stack`.
- Checkout column: single column, max-width 640px, centered; the summary card is full-width within
  it. Breakpoint: at <480px, buttons go full-width.

## Components

Format per component: anatomy → states → variants → tokens consumed → do/don't.

### Button
- **Anatomy:** container, label, optional leading icon.
- **States:** default · hover (`color-primary-hover`) · focus (2px `color-primary` ring, 2px
  offset) · active · **disabled** (muted, still legible) · **loading** (spinner replaces label,
  width locked — no reflow).
- **Variants:** primary (one per view) · secondary (outline) · destructive (`color-danger`).
- **Tokens:** `color-primary(-hover)`, `text-default`, `color-border`, `radius-control`,
  `motion-interaction`.
- **Don't:** two primaries in one view; disable without a reason the user can see.

### Text input
- **States:** default · focus (ring as Button) · **error** (`color-danger` border **plus** helper
  text — color never the only channel) · disabled · filled.
- **Tokens:** `color-surface`, `color-border`, `text-default`, `text-muted`, `color-danger`,
  `radius-control`.
- **Don't:** placeholder as label; clearing input on validation failure (`PR-checkout-guest`: a
  declined payment preserves entered data).

### Card / Order summary
- **States:** default · **empty** ("Your cart is empty" in `text-muted` + primary action back to
  shop) · **loading** (skeleton rows, no spinner-on-white).
- **Tokens:** `color-surface`, `color-border`, `radius-card`, `space-stack`.

### Inline alert
- **Variants:** danger (declined payment — `PR-checkout-guest`), success (order confirmed —
  `PR-guest-confirmation`), warning (`color-warning`).
- **States:** static; dismissible only for warning. Icon + text, never color alone.

## Accessibility rules (checkable — these become acceptance assertions)

- Text vs. surface ≥ **4.5:1**; large text and control boundaries ≥ **3:1** — declared as
  `contrastsWith` pairs in the tokens and enforced by `validate-design-tokens`.
- Focus visible on every interactive element (the ring spec above); never `outline: none` without a
  replacement.
- Touch targets ≥ 24×24 CSS px (44×44 preferred on the payment step).
- All transitions ≤ 200ms and respect `prefers-reduced-motion`.
- Every state change (error, success, loading) is announced: `aria-live="polite"` on the alert
  region; inputs tie errors via `aria-describedby`.

## Content & microcopy tone

Calm, concrete, no exclamation marks. Errors say what happened and what to do next ("Your card was
declined — try another card or check the number."), never blame ("Invalid input!").

## Requirements traceability

| Product requirement / UX state | Rendered by |
|---|---|
| `PR-checkout-guest` — guest completes payment; declined-payment path | Text input (error), Button (loading/disabled), Inline alert (danger) |
| `PR-guest-confirmation` — confirmation with order number | Inline alert (success), Card |
| `PR-order-lookup-guest` — retrieve order via email | Text input, Card (empty for "no order found") |
| Cart-empty state (product spec UX states) | Card (empty) |

Every UX state in `PRODUCT_SPEC.example.md` has a component above; none is left to per-screen
improvisation.

---
*Consumed by the build plan: the theme-generation ticket renders `design-tokens.example.yaml` into
the storefront's Tailwind config; frontend tickets cite the component contracts above in their
acceptance criteria.*
