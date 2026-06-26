# Technical Spec — Guest Checkout

> Worked example for the **technical-spec-architect** agent, designing against
> `PRODUCT_SPEC.example.md` (guest checkout). Demonstrates the inside-out discipline: architecture as
> contracts, the hard/soft constraint envelope (mirrored in `constraints.example.yaml`), the NFR
> handoff (user-facing → numeric), ADR-lite decisions, the cross-cutting checklist, and a
> requirements-traceability matrix over the `PR-*` spine.

## Summary & traced product requirements

This design satisfies **PR-checkout-guest** (place an order with no account), **PR-guest-confirmation**
(immediate confirmation + message), and **PR-order-lookup-guest** (find a prior order by email +
order number). It adds a guest-order path alongside the existing account-checkout flow, reusing the
existing cart and payment-processor integration, and introduces a guest-order lookup capability.

## Architecture overview

Three responsibilities, each a bounded component behind a contract:

- **Checkout coordinator** — orchestrates a guest order: validates the cart, captures contact /
  shipping / payment intent, delegates the charge to the payment processor, and records the order
  only on a confirmed charge. Owns the "never charge without confirming" invariant.
- **Order store** — persists guest orders and exposes lookup by `(email, orderNumber)`. Source of
  truth for order status.
- **Notifier** — sends the transactional confirmation message on a placed order.

The existing **cart** and **payment-processor adapter** are reused unchanged; this design consumes
their contracts rather than modifying them.

## Component / module boundaries & interfaces

Contracts, not code. Implementation inside each boundary is left to the build plan.

- **Checkout coordinator**
  - `placeGuestOrder(cart, contact, shipping, paymentIntent) -> OrderConfirmation | CheckoutError`
  - Guarantees: an order record exists *iff* the charge is confirmed; on any payment failure, returns
    `CheckoutError` and no order is recorded and the shopper is not charged.
- **Order store**
  - `recordOrder(order) -> orderNumber`
  - `findOrder(email, orderNumber) -> OrderView | NotFound`
  - `NotFound` is indistinguishable to the caller whether the email is unknown or merely the pair
    doesn't match (no account-existence disclosure — supports PR-order-lookup-guest).
- **Notifier**
  - `sendConfirmation(email, OrderConfirmation) -> Delivered | DeliveryFailed`
  - Delivery is best-effort and **out of band**: a `DeliveryFailed` does not roll back a placed order
    (the on-screen confirmation is the primary proof; the message is secondary).

`OrderConfirmation` carries `orderNumber`, item summary, and totals. `OrderView` carries status +
summary, never card data.

## Data model & dataflow

- **GuestOrder** — `orderNumber` (opaque, non-sequential), `email`, shipping address, line items,
  totals, status, `placedAt`. **No card data** is stored (see `pci-no-card-storage`).
- **Dataflow (place):** cart snapshot → coordinator validates → payment processor authorizes/charges
  → on confirm, `recordOrder` → return confirmation → fire-and-forget `sendConfirmation`.
- **Dataflow (lookup):** `(email, orderNumber)` → `findOrder` → `OrderView` or uniform `NotFound`.
- **Lifecycle:** guest orders retained 24 months by default (`guest-data-retention`), then purged.

## External integrations & dependencies

- **Payment processor** (existing adapter) — PCI-compliant; the system delegates the charge and never
  sees or stores raw card data.
- **Transactional message provider** (existing Notifier dependency) — used for confirmation delivery.

## Constraints

Mirrors `constraints.example.yaml`. Hard constraints bound every downstream choice; soft constraints
are defaults with escape hatches.

| key | kind | statement | owner |
|---|---|---|---|
| `deploy-aws` | hard | AWS us-east-1 primary / us-west-2 standby | platform-team |
| `lang-python` | hard | Python 3.12 for all backend services | eng-leadership |
| `compliance-soc2` | hard | Order/payment handling stays in the SOC2 boundary | security |
| `pci-no-card-storage` | hard | Never store/log card data; delegate to processor | security |
| `latency-checkout` | hard | p99 checkout-submit→confirmation < 2000 ms | eng-leadership |
| `db-postgres` | soft | Prefer Postgres unless a workload demands otherwise | data-platform |
| `boring-tech` | soft | Favor boring, well-understood tech | eng-leadership |
| `guest-data-retention` | soft | Retain guest orders 24 months by default | security |

## Cross-cutting concerns

- **Security & authz** — guest checkout is unauthenticated by design; the order store enforces the
  uniform `NotFound` so lookup can't enumerate emails. Card data never enters the boundary.
- **Observability** — emit metrics for checkout submit→confirm latency (to watch `latency-checkout`),
  charge-confirmed-without-order count (must stay 0), and confirmation delivery rate.
- **Error handling & failure paths** — payment decline: stay on payment step, preserve entered
  details, no order recorded. Processor timeout: fail closed — do **not** record an unconfirmed
  order; surface a retryable error. Notifier failure: order still placed; delivery retried out of
  band.
- **Performance (NFR handoff → numeric):** "feels instant" → **p99 submit→confirmation < 2000 ms**
  (`latency-checkout`). "Never charges without confirming" → the coordinator's record-iff-confirmed
  invariant.
- **Scaling** — checkout is write-light; the binding resource is the payment processor's rate. Lookup
  is read-light and indexed by `(email, orderNumber)`.
- **Data lifecycle** — 24-month retention then purge (`guest-data-retention`); no card data to retain.
- **Cost** — dominated by payment-processor fees; storage is negligible.

## Key decisions & alternatives (ADR-lite)

### Decision: confirmation message is best-effort, out of band
- **Context:** PR-guest-confirmation needs both an on-screen confirmation and an emailed one;
  `latency-checkout` bounds the submit→confirmation path.
- **Decision:** the on-screen `OrderConfirmation` is the primary proof and is returned synchronously;
  the emailed message is sent fire-and-forget after the order is recorded.
- **Rejected:** sending the email synchronously before returning confirmation — rejected because it
  couples the latency-critical path to a third-party message provider and risks blowing
  `latency-checkout`, and a delivery failure would wrongly fail a paid order.
- **Consequences:** delivery is retried out of band; a separate delivery-rate metric guards the
  product guardrail.

### Decision: Postgres for the order store
- **Context:** orders are transactional and must be recorded atomically with charge confirmation.
- **Decision:** use Postgres (honors the `db-postgres` soft default).
- **Rejected:** a document store — rejected; no workload need justifies leaving the default, and
  transactional guarantees favor Postgres.
- **Consequences:** none requiring an escape-hatch ADR, since we stayed on the default.

## Rollout, migration & operational concerns

- Additive: the guest path ships behind a flag alongside account checkout; no migration of existing
  orders. Roll back by disabling the flag. Lookup capability ships with it.

## Risks & open technical questions

- **Risk:** processor partial failures (charged but no confirmation returned). Mitigation: reconcile
  against the processor; never record an order without a confirmed charge, and alert on the
  charge-without-order metric.
- **Open:** exact purge mechanism for the 24-month retention — deferred to the build plan.

## Requirements-traceability matrix

| `PR-*` | Where addressed | Status |
|---|---|---|
| PR-checkout-guest | Checkout coordinator `placeGuestOrder`; record-iff-confirmed invariant; `latency-checkout` | addressed |
| PR-guest-confirmation | Synchronous `OrderConfirmation` + out-of-band Notifier (ADR) | addressed |
| PR-order-lookup-guest | Order store `findOrder` + uniform `NotFound` | addressed |

No product requirement is left unaddressed; no component traces to a requirement outside the product
spec.

---

*Next steps: the **acceptance-spec agent** turns the product spec's acceptance criteria into up-front
E2E tests; the **build-plan agent** decomposes this spec into test-first tickets that reference the
`constraints.example.yaml` keys and the `PR-*` spine above.*
