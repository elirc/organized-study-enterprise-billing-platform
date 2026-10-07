# Current System Design

## Repo Structure

- `apps/api`: Express modular monolith API.
- `apps/web`: React/Vite operational frontend.
- `apps/worker`: BullMQ background worker.
- `packages/domain`: shared billing, money, RBAC, state, and idempotency rules.
- `packages/db`: Prisma schema, client export, and seed data.
- `docs`: architecture, backlog, feature plans, journals, walkthroughs, and migration discipline.

## Major Module Connections

The frontend calls the API. The API enforces tenant context and permissions, writes database records, records audit logs, and emits outbox events. The API also queues worker jobs. The worker consumes billing jobs, writes invoices/payments, updates outbox records, and reports a heartbeat.

## Key Workflows

- Create customer: frontend action, API authorization, tenant-owned customer row, audit log.
- Create subscription: idempotent API command, transactional subscription write, audit log, outbox event, invoice job.
- Generate invoice: subscription plus plan price, coupon, tax, invoice lines, audit log, outbox event.
- Attempt payment: queued worker job, mock provider result, invoice state update, audit log, outbox event.
- Inspect operations: dashboard, audit log, health readiness, worker heartbeat.

## System Design Decisions

- Modular monolith first, because billing workflows need consistency and are easier to learn in-process.
- Integer cents for money.
- Backend RBAC as the source of truth.
- Explicit workflow states.
- Audit logs and outbox events from the beginning.
- Mock provider before real Stripe integration to make workflow behavior inspectable.

## Tradeoffs

- Development header auth is temporary.
- API tests are not yet deep enough for production confidence.
- The worker and API duplicate some invoice-generation shape; a future service extraction can reduce that.
- `db push` remains useful for local learning, while migration discipline is documented for durable changes.

## Improvements

Follow `docs/backlog.md` in order. The next highest-value product features are authentication sessions, tenant isolation guardrail tests, deeper RBAC, membership administration, and stronger billing state machines.

## The workflows, step by step (checked against this snapshot)

Line numbers refer to this snapshot. Nothing here was run.

### Every request: tenant context

1. `apps/api/src/app.ts:16` registers `attachTenantContext` for all routes.
2. `apps/api/src/shared/auth.ts:9-12` skips paths starting with `/health`.
3. `:14-15` reads `x-org-slug` and `x-user-email`, defaulting to `acme` and `owner@acme.test` (the default email comes from `config.ts:6`).
4. `:17-26` loads the membership joining that organization and user; `:28-30` throws if there is none (the error handler turns that into a 400, not a 401).
5. `:32-39` stores `organizationId`, `userId`, role and permissions on `request.tenant`.
6. Each route then calls `assertPermission(request, "...")` (`:55-62`) or, for some reads, only `requireTenant` (`:47-53`).

### Create subscription — `POST /api/billing/subscriptions`

1. Permission `billing:manage` (`billing.routes.ts:128`); body validated by `createSubscriptionSchema` (`:355-361`: seats 1-500, period 1-366 days, trial on by default).
2. `runIdempotent` (`shared/idempotent.ts:4-54`) runs the work directly if there is no `idempotency-key` header; otherwise it returns the stored response for a matching key and payload hash, rejects a key reused with a different payload, or runs the work and stores the response afterwards.
3. Inside one `prisma.$transaction` (`billing.routes.ts:136-180`): load the customer and plan scoped to the organization (`:137-145`), compute the period and trial end (`:147-149`), create the subscription as `trialing` or `active` (`:150-162`), write the audit log (`:164-171`) and the outbox event (`:173-177`).
4. After the commit, enqueue `generate-invoice` on the `billing` queue (`:182-185`).

If Redis is down, step 4 fails after step 3 committed: the subscription exists, has no invoice job, and — because `runIdempotent` stores its record only after `run` returns — a retry with the same key creates a second subscription.

### Generate invoice — worker job and manual endpoint

Two copies of the same logic: `generateInvoice` in `apps/worker/src/jobs.ts:4-105` and `createInvoiceForSubscription` in `apps/api/src/modules/billing.routes.ts:252-341`. Both:

1. load the subscription and the plan's first active price (`prices: { where: { active: true }, take: 1 }` — no ordering, so "first" is whatever the database returns);
2. take the newest active coupon and any active tax rate of the organization;
3. call `calculateInvoice` (`packages/domain/src/invoice-calculator.ts:28-49`): line subtotals, then the percentage discount, then tax on the discounted amount, each rounded to whole cents;
4. number the invoice `INV-` + (count of the organization's invoices + 1), zero-padded to six digits;
5. create the invoice as `open` with `finalizedAt` set, plus audit and outbox rows.

Only the worker checks for an existing invoice in the current period first (`jobs.ts:21-34`). Neither checks the subscription status, so a `trialing` subscription is invoiced at full price immediately.

### Attempt payment — `POST /api/billing/invoices/:id/pay` then the worker

1. The route checks `payments:manage`, confirms the invoice belongs to the organization, enqueues `attempt-payment` and answers 202 (`billing.routes.ts:232-250`).
2. `attemptPayment` (`jobs.ts:107-166`) returns early if the invoice is already `paid`; otherwise the mock provider "declines" when the total is divisible by 7 (`:120`), creates a `Payment`, marks the invoice `paid` on success, and writes audit and outbox rows.

It does not check for `void` or `uncollectible` invoices, and two queued attempts for the same invoice can both run before either marks it paid.

### Outbox

`processOutboxBatch` (`apps/worker/src/outbox-processor.ts:3-45`) runs every 5 seconds (`apps/worker/src/index.ts:42-46`). It reads up to 10 `pending` events, marks each `processing`, logs it to the console, and marks it `processed`. There is no claim step (two workers would both pick the same rows), and an event left in `processing` by a crash is never picked up again, because only `pending` rows are selected.

### Health

`/health/ready` (`app.ts:26-50`) runs `SELECT 1`, then reports the worker as `ready` if its heartbeat row is newer than 30 seconds, else `unknown`. The worker writes the heartbeat every 10 seconds (`worker/src/index.ts:48-66`). Readiness still returns HTTP 200 when the worker is `unknown`; only a database failure produces 503.

## Review findings

- **Cross-tenant payment update via the mock webhook** — see README "Known gaps". The fix is to look the payment up with `organizationId` (or, for a real provider, to verify a signature and derive the organization from the stored payment, never from headers).
- **Webhook events are global.** `providerEventId` is unique across all organizations (`packages/db/prisma/schema.prisma`, `WebhookEvent`), and `GET /api/payments/events` also returns events whose `organizationId` is null (`payments.routes.ts:15-21`).
- **Not-found is reported as 400.** `findFirstOrThrow` errors fall through to the generic handler (`app.ts:58-62`).
- **The portal shows one customer to every member.** `GET /api/platform/portal` returns the organization's newest customer (`platform.routes.ts:36-55`); nothing links a user to a customer.
- **Coupons and taxes are organization-wide** (see the invoice steps above).
- **State machine helpers are unused** outside `domain.test.ts`.
- **Test coverage** is five domain tests (`packages/domain/src/domain.test.ts`). None touches the API, worker, database or tenant isolation.

## Exercises

Each has a **Goal** and a **Check** you can observe.

1. **Trace a subscription to a paid invoice.**
   *Goal:* with Postgres, Redis, API and worker running and the seed loaded, click "create subscription" in the web app for the Pro plan (2900 cents, 3 seats, 10% coupon, 825 basis-point tax), wait a few seconds, then queue a payment.
   *Check:* the new invoice has subtotal 8700, discount 870, tax 646 and total 8476 (all in cents); 8476 is not divisible by 7, so the payment succeeds and the invoice becomes `paid`. `GET /api/platform/outbox` shows `billing.subscription.created`, `billing.invoice.generated` and `payment.succeeded` events moving to `processed`.
2. **Prove the duplicate-invoice gap.**
   *Goal:* after exercise 1, call `POST /api/billing/subscriptions/<id>/invoices` for the same subscription.
   *Check:* a second invoice is created for the same period with the next `INV-` number. Then add the worker's period check to `createInvoiceForSubscription` (better: move the logic into one shared function) and confirm the second call returns the existing invoice.
3. **Scope the webhook.**
   *Goal:* change the webhook to `findFirst({ where: { id: paymentId, organizationId } })` and return 404 when nothing matches.
   *Check:* a request with `x-org-slug` set to another organization (create one in the seed) and a payment ID from `acme` returns 404, and the payment's status is unchanged.
4. **Wire a state machine in.**
   *Goal:* use `transitionInvoice(invoice.status, "paid")` in `attemptPayment` and in the webhook before updating.
   *Check:* queueing a payment for an invoice you set to `void` in Prisma Studio now fails the job with "Cannot transition invoice from void to paid." instead of creating a successful payment.
