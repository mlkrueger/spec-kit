# Repo Map — Storefront (adding: Guest Checkout)

> Worked example of the Phase-0 brownfield survey (`reference/brownfield.md`). Captures the existing
> repo once so the four architect agents add the **guest-checkout** feature *to* it without inventing
> what already exists. Feature slug: `guest-checkout`.

## Stack(s)

- **`python`** — backend services (FastAPI, Python 3.12), tests with pytest.
- **`js-frontend`** — storefront UI (SvelteKit, Svelte 5), Vitest + Playwright.

(De-facto language/runtime constraints: Python 3.12 backend, SvelteKit frontend — see *Inherited
constraints*.)

## Architecture & module seams

- `src/checkout/` — existing **account checkout** coordinator. The guest path attaches here.
- `src/orders/` — existing order persistence (`store.py`), Postgres-backed. The new guest-order record
  and lookup extend this.
- `src/payments/` — existing **payment-processor adapter** (PCI-compliant). Reuse unchanged.
- `src/notify/` — existing transactional message sender. Reuse for guest confirmation.
- `web/src/routes/checkout/` — existing checkout UI; a guest variant attaches here.

## Test harness

- **Backend:** `pytest`; `tests/{unit,integration,e2e}/`; fixtures in `conftest.py`. Single test:
  `pytest tests/unit/test_x.py::test_y`.
- **Frontend:** `vitest run` (component tests `*.component.test.ts`, jsdom); **existing E2E harness** at
  `e2e/support/harness.ts` with `playwright test`, auth bypass in `e2e/fixtures/`. **Reuse this rig** —
  do not build a new one.
- Naming infixes already in use: `*.component.test.ts` (DOM), `*.e2e.ts` (Playwright),
  `*_integration` for backend integration tests.

## Conventions (inherited)

- Dataclasses for domain models, Pydantic for API schemas (not mixed).
- Errors fail closed on authz; the order write is record-iff-charge-confirmed.
- Observability: structured logs + per-request metrics already wired.

## Existing feature areas

- Account signup/login, account checkout, order history (account-only), payments, notifications.
- Guest checkout's boundary: it must **not** modify account checkout, login, or the existing order-history
  views — those are existing behavior the new feature integrates beside, not into.

## Existing PR-*/ticket-key namespace

- Defined under `features/account-checkout/`: `PR-account-checkout-*`, keys `account-checkout-*`.
- No `guest-checkout` IDs exist yet → the slug is free. Mint `PR-guest-checkout-*` and keys
  `guest-checkout-*`.

## Inherited constraints (→ hard constraints in constraints.yaml)

- `lang-python` — Python 3.12 backend (existing).
- `db-postgres` — Postgres for orders (existing; here it is effectively hard, not just a default).
- `pci-no-card-storage` — payments delegated to the existing PCI adapter; never store card data.
- `web-sveltekit` — frontend is SvelteKit/Svelte 5 (existing).

## Integration points & regression surface

- **Integration:** `src/checkout/` (add guest coordinator beside account), `src/orders/store.py` (add
  guest-order record + lookup), `web/src/routes/checkout/` (guest UI variant).
- **Regression surface to keep green:** existing `tests/integration/test_account_checkout*`, the
  `e2e/account-checkout-*.e2e.ts` journeys, and `src/orders/` unit tests. The done-gate must include
  these staying green.
