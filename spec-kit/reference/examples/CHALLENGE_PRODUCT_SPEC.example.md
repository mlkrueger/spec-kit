# Challenge Report — Product Spec, Guest Checkout

> Worked example for the **spec-challenger** agent, challenging `PRODUCT_SPEC.example.md`.
> Demonstrates the report format from `challenge-standards.md`: all three severities, the full
> disposition range (a resolved blocker, a rebuttal that stands, a question that rides to the
> human), and the certification note that makes attacked-and-held distinguishable from unexamined.
> Note what is *absent*: no replacement prose, no edits to the spec — findings, never fixes.

---

**Artifact:** `PRODUCT_SPEC.md` (guest checkout) — challenged as approved-candidate before the
phase-1 checkpoint.
**Passes:** 2 (full loop).
**Verdict:** `revised-and-certified` — 1 resolved, 1 rebutted (stands), 1 question for the human.
No findings contested.

## Findings

### CH-product-1 — `blocker` — resolved

- **Target:** `PR-checkout-guest` × `PR-order-lookup-guest`.
- **Claim:** The two requirements state contradictory access rules for a guest's past order —
  `PR-checkout-guest` promises the order is "retrievable later via the email they provided"
  (email alone), while `PR-order-lookup-guest` requires email **and** order number.
- **Failure scenario:** The acceptance-spec-architect writes the checkout journey asserting
  retrieval by email alone (as `PR-checkout-guest` reads), while the build plan implements lookup
  requiring both values (as `PR-order-lookup-guest` reads). The acceptance journey fails against a
  correctly built system — or worse, the builder "fixes" it by allowing email-only enumeration of
  orders, which `PR-order-lookup-guest`'s anti-disclosure clause exists to prevent.
- **Disposition (pass 2):** `resolved`. `PR-checkout-guest` now reads "retrievable later using the
  email and order number from their confirmation," consistent with the lookup requirement and the
  anti-disclosure clause.

### CH-product-2 — `major` — rebutted, stands

- **Target:** UX flows & states; the requirement set around `PR-order-lookup-guest`.
- **Claim:** The spec is silent on the mistyped-email path: a guest who typos their email at
  checkout receives no confirmation and — because the order number only arrives by email — has no
  lookup path at all. No requirement, state, or non-goal covers recovery.
- **Failure scenario:** Downstream phases treat the silence as "not a case": no acceptance journey
  exercises it, no state is designed for it, and the first support ticket becomes an unplanned
  scope debate mid-build.
- **Architect rebuttal (verbatim):** "Deliberate v1 scope: recovery for a mistyped email requires
  identity verification we explicitly don't have for guests (that's what accounts are for — see
  non-goals). Support handles the rare case manually via payment-reference lookup. Added the
  explicit non-goal 'no self-service recovery for orders placed with a mistyped email' so the
  boundary is visible rather than silent."
- **Disposition (pass 2):** `rebutted — stands`. The rebuttal holds: the gap was silence, and the
  new non-goal converts it into a stated boundary downstream phases can see. No journey or
  component is now owed for it.

### CH-product-3 — `question` — for the human

- **Target:** `PR-order-lookup-guest` (anti-disclosure clause) × the infrequent-returning-shopper
  persona.
- **Claim:** The neutral "no matching order" response protects against email enumeration but gives
  a legitimate shopper who *knows* they ordered no way to distinguish "wrong order number" from
  "wrong email" — a genuine privacy-versus-recoverability trade-off the spec resolves toward
  privacy without recording that the trade-off was chosen.
- **Failure scenario:** If unstated, a future iteration "improves UX" by making the message more
  helpful and silently reverses a security posture the business may have depended on.
- **Disposition:** `open` — routed to the checkpoint, not the architect. If privacy-first is the
  intent, one sentence recording the choice closes this.

## Certification note (attacked and held)

- **Mechanism filter:** hunted for solutions in disguise across all three `PR-*`s and the flows —
  held; behaviors are stated as trigger → user-visible outcome with no machinery (payment
  processing, order storage, and email delivery are all described by their observable effects).
- **NFR handoff:** checked every user-facing NFR for smuggled numbers — held; targets are
  deferred to the technical spec.
- **Non-goals:** attacked for the classic scope leaks around guest checkout (account merging,
  order editing, payment-method changes) — held; each is an explicit non-goal (and CH-product-2's
  outcome added one more).
- **States:** empty, loading, error, and the lookup-miss edge are described (not designed) — held.
- **Personas:** each carries a real job; the excluded power-user segment is named — held.

*Report ends. The artifact was not modified by this review; the checkpoint presents this report
alongside it, CH-product-3 first.*
