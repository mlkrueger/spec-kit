# Style Tile — Acme Storefront (guest checkout)

> Example style tile for the guest-checkout example feature, produced by **design-spec-architect**.
> **Mode: reference-controlled** — extracted from `acme-brand-guide.pdf` + the storefront's existing
> `tailwind.config.js`; this tile is a *distillation* of those sources, and the approval question is
> **"did we extract your brand correctly?"** Approved before `UI_STYLE_GUIDE.example.md` and
> `design-tokens.example.yaml` were derived from it.

## Brand atmosphere

**Assured · unhurried · precise · warm-neutral**

Checkout is where a guest decides whether to trust a store they have no account with
(`PR-checkout-guest`). Everything below serves *assured*: generous whitespace, one accent used
sparingly, no visual noise competing with the order summary. *(Source: brand guide §1, "calm
commerce".)*

## Color palette — roles, not hexes

| Role | Value | Used for | Source |
|---|---|---|---|
| `color-primary` | Acme Indigo `#4f46e5` | The one accent: Place order, Continue | brand guide §2.1 |
| `color-surface` | `#f8fafc` | Page and card background | tailwind.config.js |
| `text-default` / `text-muted` | `#0f172a` / `#64748b` | Body / helper copy | brand guide §2.3 |
| `color-border` | `#e2e8f0` | Hairlines on inputs, cards | tailwind.config.js |
| `color-danger` / `color-success` | `#dc2626` / `#15803d` | Declined payment / order confirmed | brand guide §2.4 |

Light theme only for this feature (the storefront has no dark mode — inherited reality, not a
choice made here).

## Typography

- **One family** (the storefront's existing Inter stack), weights 400/600 only. *(brand guide §3)*
- Scale: 14 / 16 / 18 / 20 / 24 px — body is 16, one heading level per view at 24.
- Example rhythm: **Order summary** (24/600) → "3 items · $86.40" (16/400) →
  *"Prices include tax."* (14/400, muted).

## Core UI elements

- **Primary button** — indigo fill, white label, `radius-control`; hover darkens one step; disabled
  is muted, never invisible. One per view.
- **Secondary button** — outline on `color-border`, `text-default` label.
- **Input** — surface fill, hairline border, visible focus ring (`color-primary`, 2px offset);
  error state pairs `color-danger` border **with** helper text — color is never the only channel.
- **Card** — `radius-card`, hairline border, no shadow (brand guide: "no elevation theater").

## Example micro-layout *(illustration only — non-binding)*

Order-summary card: heading, three line items in muted text, hairline divider, total in 18/600,
full-width primary button. Proves the palette, rhythm, and elements hang together.

## Imagery & iconography

Product photos on white, no filters, hairline-framed. Icons: 1.5px stroke, rounded caps, line style
only — never filled. *(brand guide §5)*

## Anti-patterns — what Acme is not

- No gradients, no drop shadows, no elevation stacks. *(brand guide §4)*
- Never pure black text; never more than one accent per view.
- No countdown timers or urgency banners in checkout — *assured, not pressured*
  (`PR-checkout-guest` non-goal: no dark patterns).

---
*Approve this tile before the UI style guide and tokens are derived. Tone changes are cheap here,
expensive after systematization.*
