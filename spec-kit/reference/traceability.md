# Cross-Phase Traceability Spine (Reference)

> Shared convention across every `spec-kit` phase — the product spec, the technical spec, and the
> downstream `build-plan` and `acceptance-plan`. It defines the stable IDs that turn the
> spec-phase documents into one **linked, auditable chain**: idea → requirement → design decision →
> buildable/testable ticket.

## Why a spine

Autonomous agent hand-offs are only trustworthy if you can mechanically answer: *"Is every product
requirement designed, built, and tested?"* A stable ID per requirement makes that a lookup instead
of a judgment call. It reuses the same "reference-by-stable-key" idea the `ticket-plan` already
relies on for `parent`/`blockedBy`, extended across phases.

## The ID: `PR-<slug>`

Every product requirement gets a **stable, unique, kebab-case** identifier, prefixed `PR-`
(**P**roduct **R**equirement):

```
PR-checkout-guest
PR-order-history
PR-password-reset
```

Rules:

- **Stable.** Once assigned, an ID never changes meaning. If a requirement is split, the original ID
  retires and two new ones are minted (note the split); never silently repurpose an ID.
- **Unique** across the product spec.
- **Kebab-case**, matching the `key` pattern used everywhere else (`^PR-[a-z0-9]+(-[a-z0-9]+)*$`).
- **Slug is descriptive**, not numeric — `PR-checkout-guest`, not `PR-001`. Numeric IDs drift out of
  sync with meaning as requirements churn; descriptive slugs survive reordering.

### Feature-scoped IDs (brownfield / multi-feature repos)

When adding features to an existing repo (see `brownfield.md`), each feature lives under
`features/<feature-slug>/` and **prefixes its IDs with the feature slug** so they never collide across
features:

```
PR-guest-checkout-place-order
PR-guest-checkout-order-lookup
```

This is still valid kebab-case, so every schema and validator accepts it unchanged — feature scoping is
convention, not new machinery. Ticket and milestone `key`s are prefixed the same way
(`<feature>-<ticket>`), which keeps the tracker stamps derived from them collision-free too. Before
minting any ID/key, check it against the existing namespace recorded in `REPO_MAP.md`.

## How each phase references the spine

| Phase | Artifact | How it uses `PR-*` |
|---|---|---|
| Product | `PRODUCT_SPEC.md` | **Defines** each `PR-*` on its requirement / user story. The source of truth. |
| Technical | `TECHNICAL_SPEC.md` | Its **requirements-traceability matrix** maps every `PR-*` → where the design addresses it, flagging any unaddressed. Constraints may reference the `PR-*` that motivates them. |
| Acceptance | `acceptance-plan.yaml` | Each E2E ticket's `tracesTo` lists the `PR-*` journeys it exercises. |
| Build | `build-plan.yaml` | Each implementation ticket's `tracesTo` lists the `PR-*` it helps satisfy. |

## What this buys you

- **Coverage as a query.** "Which requirements have no design?" → `PR-*` in the spec but absent from
  the tech-spec matrix. "Which have no test?" → absent from every `tracesTo`. The downstream
  validators (`validate-build-plan`, `validate-constraints`) enforce that referenced IDs actually
  exist, so the spine is *checked*, not aspirational.
- **Blast-radius answers.** Change `PR-checkout-guest` → grep the chain for every design decision and
  ticket that traces to it.
- **Clean hand-offs.** Each agent can verify its own input was fully consumed before producing its
  output.

## Discipline

- Mint the ID **when you write the requirement**, in the product spec — never retrofit later.
- Keep the slug aligned with the requirement's meaning; if the meaning changes materially, that's a
  split (retire + mint), not an edit.
- Downstream agents **reference** `PR-*` IDs; they never invent new ones. A ticket that can't trace
  to any requirement is either missing a requirement upstream (flag it) or is out of scope.
