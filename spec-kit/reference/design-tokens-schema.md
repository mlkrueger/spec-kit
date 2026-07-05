# Design Tokens Schema (the machine-readable design contract)

> The **design-token set** is the machine-readable hand-off between the `design-spec-architect`
> agent (producer) and every downstream consumer — the `build-plan-architect` (frontend tickets) and
> the theme-generation ticket that turns tokens into Tailwind config / CSS custom properties. The
> architect emits `design-tokens.yaml` beside `UI_STYLE_GUIDE.md`; the guide's token section is the
> human mirror of this file.
>
> **The file holds the token decisions only.** The atmosphere narrative lives in `STYLE_TILE.md`;
> the component contracts and usage rules live in the guide; this file is the structured, lintable
> form the theme is generated from.

The machine-validatable definition is `design-tokens.schema.json` (JSON Schema draft 2020-12). This
document is the prose companion: the object model, the constraints JSON Schema can't fully express,
and the consumption contract. When the two disagree, the JSON Schema wins for field shape; this doc
wins for *meaning and process*. The format is deliberately **DTCG-adjacent, lighter** — portable
into Style Dictionary / Tailwind generators without committing to the full `$value`/`$type`
spelling.

## Object model

```
token set
├── version            # const 1
├── spec               # companion doc filename (UI_STYLE_GUIDE.md)
├── mode?              # invention | reference (where authority comes from)
├── references?        # reference mode: the controlling sources, precedence order
├── primitives?        # tier 1: category → { name → literal }, no meaning attached
└── tokens[]           # tier 2: the flat semantic-role list
    └── token
        ├── key            # REQUIRED. stable, kebab-case, UNIQUE across the file
        ├── category       # REQUIRED. color|typography|space|radius|border|shadow|motion|z-index|breakpoint
        ├── value          # REQUIRED. '{primitives.<cat>.<name>}' alias (preferred) or literal
        ├── description    # REQUIRED. the role in one line
        ├── source?        # provenance — expected on every token in reference mode
        ├── contrastsWith? # color: the token key this one must stay legible against
        ├── minContrast?   # the WCAG ratio the pair must meet (default 4.5; requires contrastsWith)
        └── tracesTo?      # optional PR-* ids this token serves
```

## Two tiers — the load-bearing distinction

- **Primitives** are the raw scales (`color.indigo-600`, `space.4`) with no meaning attached; they
  exist so semantic tokens have something stable to alias and a re-theme has one place to swap.
- **Semantic tokens** are the roles components actually reference (`color-primary`, `text-muted`,
  `radius-card`). **Components reference semantic tokens only** — never primitives, never raw
  hex/px. That rule (enforced by the token-conformance lint on frontend tickets) is what makes a
  re-theme a token swap instead of a codebase grep.

## Constraints not enforceable in JSON Schema

`validate-design-tokens` must check these in addition to the JSON Schema:

1. **`key` uniqueness** — every `token.key` is unique across the file.
2. **Alias resolution** — every `{primitives.<category>.<name>}` value resolves to a declared
   primitive. A dangling alias is a broken theme.
3. **The required role floor** — the semantic color roles a usable UI cannot lack (see
   `design-spec-standards.md`): `color-primary`, `color-surface`, `text-default`, `text-muted`,
   `color-border`, `color-danger`, `color-success`.
4. **Contrast** — for every token with `contrastsWith`: the referenced key exists, both sides
   resolve to computable hex colors, and the computed WCAG ratio meets `minContrast` (default 4.5).
   This makes the accessibility floor *mechanically enforced*, the highest-leverage check in the
   file.
5. **Traceability existence** — when any `tracesTo` is used, a product spec is required (a sibling
   `PRODUCT_SPEC.md` or `--product-spec`) and every referenced id must appear in it. A token set
   with no `tracesTo` at all validates without one (the core role floor serves every UI, not one
   requirement).
6. **Provenance in reference mode** — when `mode: reference`, every semantic token should carry a
   `source`; missing ones are reported (warnings, not failures — the split between extracted and
   derived must stay visible).

## Consumption contract (how downstream consumers use this)

- The **build-plan-architect** (for frontend features) emits an early foundational ticket —
  *generate the theme from `design-tokens.yaml`* (Tailwind config / CSS custom properties) plus the
  non-shipping `/style-tile` reference page — that later frontend tickets are `blockedBy`. Each
  frontend ticket cites the guide's component contract and carries token-conformance + the guide's
  accessibility rules in its `acceptanceCriteria`.
- A **token-conformance lint** on built UI rejects hardcoded color/spacing values, pointing
  offenders at the semantic role to use instead.
- The `PR-*` ids in `tracesTo` share the same spine as every other artifact (`traceability.md`).

## Validation

```sh
# resolves a sibling PRODUCT_SPEC.md automatically when tracesTo is used:
${CLAUDE_PLUGIN_ROOT}/bin/validate-design-tokens design-tokens.yaml
# or point at a product spec elsewhere:
${CLAUDE_PLUGIN_ROOT}/bin/validate-design-tokens design-tokens.yaml --product-spec ../PRODUCT_SPEC.md
```

It runs the JSON Schema plus the non-schema checks above — including computing the real WCAG
contrast ratios from resolved hex. The design-spec-architect refuses to hand over a token file that
fails validation.

## Example

See `examples/design-tokens.example.yaml` for a complete, valid token set exercising both tiers,
the role floor, `contrastsWith` pairs, reference-mode provenance, and the `tracesTo` spine.
