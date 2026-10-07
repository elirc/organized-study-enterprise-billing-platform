# Walkthrough: `apps/api/src/modules/billing.routes.ts`

## What The File Does

This router owns customer, subscription, invoice, and payment-queue API workflows.

## Why It Exists

Billing is a central module with high coordination cost. Keeping its routes together makes it easier to trace workflow state while the project is still a modular monolith.

## How It Connects

- Uses `assertPermission` for backend authorization.
- Uses Prisma for tenant-scoped reads and writes.
- Uses `runIdempotent` for subscription creation.
- Uses domain invoice calculation.
- Writes audit logs and outbox events.
- Queues worker jobs through BullMQ.

## Important Sections

- `GET /dashboard`: aggregates operational metrics.
- `POST /customers`: creates tenant-owned customers.
- `POST /subscriptions`: creates subscriptions transactionally and queues invoice work.
- `createInvoiceForSubscription`: turns subscription, price, coupon, and tax records into invoice rows.
- `POST /invoices/:invoiceId/pay`: queues a payment attempt.

## Senior-Level Concepts

This file shows why route handlers need boundaries. A single business action can require validation, authorization, tenant scoping, transactions, audit logs, idempotency, and async jobs. When this file grows, the next refactor should extract billing services while keeping the same workflow shape.

## How To Read It

Trace `POST /subscriptions` first. It is the best example of a real backend workflow in the initial system.

## Line by line (this snapshot)

| Route | Lines | Permission | Notes |
|---|---|---|---|
| `GET /dashboard` | 14-51 | `billing:read` | open-invoice and successful-payment totals are computed from the **latest 8** invoices and payments only (`:25-35`, `:42-45`), so they are not organization totals |
| `GET /customers` | 53-78 | `billing:read` | optional case-insensitive search on name and email; 50 newest |
| `POST /customers` | 80-108 | `billing:manage` | customer create and audit log are **two separate writes** (`:86-104`), not one transaction; no outbox event |
| `GET /subscriptions` | 110-123 | `billing:read` | 50 newest |
| `POST /subscriptions` | 125-193 | `billing:manage` | idempotent when an `idempotency-key` header is sent; transaction plus queued job |
| `GET /invoices` | 195-216 | `billing:read` | `status` query string is cast with `as never` (`:203`); an unknown value makes Prisma throw, which becomes a 400 |
| `POST /subscriptions/:id/invoices` | 218-230 | `billing:manage` | calls `createInvoiceForSubscription` with no period check |
| `POST /invoices/:id/pay` | 232-250 | `payments:manage` | enqueues only; answers 202 |

`createInvoiceForSubscription` (`:252-341`) is described step by step in `docs/system-design-current.md`.

## Look for

- **Coupon and tax are chosen per organization, not per customer** (`:274-280`). Deactivate `WELCOME10` and every future invoice loses its discount; add a second active coupon and it silently replaces the first.
- **Invoice numbers race.** `count + 1` (`:294-300`) gives two concurrent invoices the same number; the unique index on `(organizationId, number)` makes the second one fail.
- **Trials are invoiced.** A subscription created as `trialing` is invoiced at full price at once — by the queued worker job, and again if someone uses the manual endpoint.
- **`currency: price.currency as "USD"`** (`:282`) silences the type checker; a price in a currency the domain does not know would still flow through.

## Exercises

1. **Goal:** create two subscriptions in quick succession with the same `idempotency-key` header and the same body (two terminals, or `Promise.all` in a script). **Check:** count the subscriptions for that customer before and after: there may be two new rows instead of one. Explain from `shared/idempotent.ts:15-51` why, and propose inserting the key row *before* running the work.
2. **Goal:** make `POST /customers` write the customer and audit log in one `prisma.$transaction`, plus a `billing.customer.created` outbox event. **Check:** with the audit write temporarily changed to throw, no customer row is left behind.
3. **Goal:** compute the dashboard totals with `aggregate` (`_sum: { total: true }` filtered by status) instead of from the latest 8 rows. **Check:** after creating nine open invoices, `openInvoiceTotal` equals the sum of all nine.
