# Product Spec — Guest Checkout

> Worked example for the **product-spec-architect** agent. Demonstrates the outside-in discipline:
> observable behaviors, `PR-*` IDs, first-class non-goals, Given/When/Then acceptance criteria, and
> user-facing NFRs with their numbers deliberately deferred to the technical spec.

## Summary

Let shoppers buy what's in their cart without creating an account, and let them find that order
again later using only the email they entered. The goal is to remove the account-creation step as a
barrier to a first purchase, while still giving the shopper a way back to their order.

## Problem & context

Today a shopper cannot pay until they create an account. Analytics show a large drop-off on the
account-creation step of the funnel: many shoppers who have added an item and reached checkout
abandon there rather than register. Support also fields recurring requests from one-time buyers who
do not want "yet another login." The pain is concentrated in first-time and infrequent shoppers; the
business loses completed purchases it has otherwise already earned.

Evidence: funnel drop-off at the account step; a cluster of support tickets requesting "buy without
an account"; comparable storefronts offer guest checkout as standard.

## Goals & non-goals

**Goals**
- Let a shopper complete a purchase without creating an account.
- Let a guest find and view their past order later using the email they provided.
- Reduce abandonment at the checkout/account step.

**Non-goals** (explicit, first-class)
- **Not** building account creation, login, or password management in this work.
- **Not** offering guest order *editing or cancellation* — view/lookup only.
- **Not** merging a guest's history into an account if they register later (a possible future
  follow-up, explicitly out of scope here).
- **Not** changing the payment methods on offer.

## Personas

- **First-time shopper (primary).** Job: buy this one item now with the least friction; does not
  want a relationship with the store yet.
- **Infrequent returning shopper.** Job: occasionally re-buys; remembers their email but not a
  password; wants to check on a past order.
- **Explicitly not for:** high-frequency power users who want saved carts, order history dashboards,
  and one-click reorder — those are served by accounts, which this work does not touch.

## User stories

- **PR-checkout-guest** — As a first-time shopper, I want to pay for the items in my cart without
  creating an account, so that I can complete my purchase with minimal friction.
- **PR-order-lookup-guest** — As a guest who has ordered before, I want to look up my past order
  using my email and order number, so that I can check its status without an account.
- **PR-guest-confirmation** — As a guest, I want an immediate confirmation of my order, so that I
  have proof of purchase and a reference for later lookup.

## Functional requirements (observable behaviors)

- **PR-checkout-guest** — *When* a shopper with at least one item in their cart chooses to check out
  as a guest and provides valid contact, shipping, and payment details, *then* they can complete the
  purchase and are shown an order confirmation; the order is recorded and retrievable later via the
  email they provided. No account is created.
- **PR-guest-confirmation** — *When* a guest's order is successfully placed, *then* they immediately
  see a confirmation showing an order number and a summary, and a confirmation message is sent to the
  email they entered.
- **PR-order-lookup-guest** — *When* a guest provides the email and order number from a prior guest
  order, *then* they are shown that order's current status and summary; *when* either value does not
  match a known order, *then* they are told the lookup did not match, without revealing whether the
  email exists.

## UX flows & states

- **Happy path:** cart → "check out as guest" → contact/shipping → payment → confirmation.
- **Empty:** an empty cart cannot enter guest checkout; the shopper is returned to the cart with a
  prompt to add items.
- **Loading:** while payment is being confirmed, the shopper sees a clear in-progress state and
  cannot double-submit.
- **Error:** a declined payment keeps the shopper on the payment step with a plain-language reason and
  their entered details preserved; a failed order placement never charges the shopper without a
  confirmation.
- **Edge — lookup miss:** an email/order-number pair that doesn't match shows a neutral "no matching
  order" message (no account-existence disclosure).

*(Any wireframe attached to this spec is non-binding illustration, not design.)*

## Acceptance criteria (Given/When/Then)

- **PR-checkout-guest** — *Given* a shopper with one item in their cart who has chosen guest
  checkout, *when* they submit valid contact, shipping, and valid payment details, *then* they see an
  order confirmation with an order number, the order is recorded against their email, and no account
  exists for them afterward.
- **PR-guest-confirmation** — *Given* a successfully placed guest order, *when* the confirmation
  screen appears, *then* it shows the order number and an item/total summary, and a confirmation
  message is delivered to the entered email.
- **PR-order-lookup-guest** — *Given* a prior guest order, *when* the guest enters its email and order
  number, *then* the order's status and summary are shown; *given* a non-matching pair, *when* it is
  submitted, *then* a neutral no-match message is shown and account existence is not revealed.

## Success metrics & guardrails

- **Primary success metric:** checkout completion rate for shoppers who reach the checkout step
  (expected to rise as the account barrier is removed).
- **Guardrail:** payment dispute / chargeback rate must not rise materially versus the
  account-checkout baseline.
- **Guardrail:** confirmation-message delivery success stays at or above the current transactional
  baseline.

## User-facing non-functional requirements

*(Numbers deliberately deferred to the technical spec — see the NFR handoff.)*

- **Feels instant** — submitting a guest order returns a confirmation quickly enough that the shopper
  never wonders whether it worked.
- **Never charges without confirming** — a shopper is never left charged but unconfirmed.
- **Trustworthy with data** — a guest's contact and order details are handled securely and not kept
  longer than needed.
- **Accessible** — the entire guest-checkout flow is usable with assistive technology and the
  keyboard alone.

## Open questions & assumptions

- *Assumption:* guests provide a working email; deliverability failures are surfaced but out of scope
  to remediate here.
- *Open:* should a guest be offered (not required) account creation *after* confirmation? Deferred —
  out of scope for this document.

## Out of scope for this document: implementation

This spec describes only *what* guest checkout does and *why*. How it is built — services, datastore,
payment integration, latency budgets, retention windows — is decided in `TECHNICAL_SPEC.md` by the
technical-spec-architect, which will also translate each user-facing NFR above into a numeric target.
