# Walkthrough: `packages/db/prisma/schema.prisma`

## What The File Does

This file defines the durable data model for the platform. It is the source of truth for tenant-owned data, billing workflow state, auditability, idempotency, and worker coordination.

## Why It Exists

Enterprise billing systems are mostly about preserving correct state over time. The schema makes important business concepts explicit instead of hiding them in unstructured JSON or route logic.

## How It Connects

- `packages/db/src/index.ts` exports the Prisma client used by API and worker code.
- `apps/api/src/modules/*.routes.ts` query tenant-owned rows through this schema.
- `apps/worker/src/jobs.ts` creates invoices and payments from subscription records.
- `packages/domain` helpers calculate money and validate state, but persistence happens here.

## Important Sections

- Organization, User, Membership, Team: tenant identity and RBAC foundation.
- Plan, Price, Entitlement, Coupon, TaxRate: product catalog.
- Customer, Subscription, Invoice, InvoiceLine, Payment: billing workflow.
- WebhookEvent, IdempotencyKey: retry-safe external command handling.
- AuditLog, OutboxEvent, WorkerHeartbeat: operational traceability.

## Senior-Level Concepts

Notice that most tenant-owned models include `organizationId`. This is not just a convenience field; it is a safety boundary. Also notice explicit enums for subscription, invoice, payment, and outbox status. Real systems avoid many booleans because boolean combinations create ambiguous states.

## How To Read It

Start with `Organization`, then follow relations outward. Ask which rows belong to a tenant, which rows are append-only, and which rows represent workflow state.

## Line by line (this snapshot)

`packages/db/prisma/schema.prisma` has 21 models and 6 enums.

| Area | Models (line) | Uniqueness that matters |
|---|---|---|
| Tenancy | `Organization` (53), `User` (71), `Membership` (80), `Team` (94), `TeamMember` (105), `UserPreference` (322) | `Organization.slug`; `User.email` is global; one membership per `(organizationId, userId)` |
| Catalog | `Plan` (114), `Price` (132), `Entitlement` (146), `Coupon` (172), `TaxRate` (184) | plan code, coupon code and tax-rate name unique per organization |
| Billing | `Customer` (156), `Subscription` (195), `Invoice` (216), `InvoiceLine` (241), `Payment` (251) | customer email and invoice number unique per organization |
| Retry safety | `WebhookEvent` (271), `IdempotencyKey` (282) | `providerEventId` unique **globally**; idempotency key unique per `(organizationId, key)` |
| Operations | `AuditLog` (293), `OutboxEvent` (307), `WorkerHeartbeat` (334) | heartbeat unique per worker name |

`packages/db/src/index.ts` re-exports everything from `@prisma/client` and keeps one `PrismaClient` on `globalThis` outside production, with query logging when `NODE_ENV=development`.

## Look for

- **`Invoice.status` defaults to `draft`**, but both invoice writers create invoices directly as `open` with `finalizedAt` set; the `draft → open` step in `billing-state.ts` never happens.
- **Nullable organization on retry tables.** `WebhookEvent.organizationId` and `IdempotencyKey.organizationId` are optional. With a NULL organization, PostgreSQL treats the composite unique key on `IdempotencyKey` as non-conflicting.
- **`Payment` has no uniqueness on `providerRef`**, and nothing stops two `succeeded` payments for the same invoice.
- **No foreign key from `Payment.organizationId` to the invoice's organization.** The schema allows a payment row whose organization differs from its invoice's; only code keeps them aligned, and the mock webhook shows how that can slip (`payments.routes.ts:51-59`).
- **`Subscription.canceledAt` exists but no code sets it**; there is no cancel endpoint.

## Exercises

1. **Goal:** list every model without an `organizationId` column and write down which parent provides its tenant. **Check:** your list is `User`, `TeamMember`, `Price`, `Entitlement`, `InvoiceLine`, `WorkerHeartbeat` (plus the two nullable ones), and only `User` and `WorkerHeartbeat` are global by design; the other four inherit their tenant from a parent row.
2. **Goal:** add a partial unique index (in a migration, since Prisma cannot express it in the schema) so an invoice can have at most one `succeeded` payment. **Check:** inserting a second succeeded payment for the same invoice in Prisma Studio fails.
3. **Goal:** open Prisma Studio after the seed. **Check:** you find one organization (`acme`), one user (`owner@acme.test`), two plans (`pro` at 2900 cents, `enterprise` at 9900 cents), one coupon (`WELCOME10`), one tax rate (825 basis points) and one customer.
