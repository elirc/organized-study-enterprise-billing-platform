# Walkthrough: `apps/worker/src/jobs.ts`

## What The File Does

This file implements asynchronous billing jobs: generating invoices and attempting mock payments.

## Why It Exists

Not every billing task belongs in the HTTP request. Workers let the API respond quickly while slower or retryable work continues separately.

## How It Connects

- API routes enqueue jobs through BullMQ.
- Worker jobs read and write Prisma records.
- Domain invoice calculation keeps money math consistent with API behavior.
- Outbox events record follow-up events for future notification and integration features.

## Important Sections

- `generateInvoice`: ensures only one invoice is generated for a subscription period, calculates totals, writes invoice lines, audit logs, and outbox events.
- `attemptPayment`: creates a deterministic mock payment and updates invoice state.

## Senior-Level Concepts

The worker checks for an existing invoice in the subscription period. This is a practical idempotency guard for retried jobs. It is not enough for every production race, but it teaches the instinct: background jobs must assume retries.

## How To Read It

Read each job as a transaction. Ask what happens if the process crashes halfway, what can be retried, and what audit trail remains.

## Line by line (this snapshot)

- `apps/worker/src/index.ts:8-16` validates `REDIS_URL` and opens the Redis connection; `:18-32` one BullMQ `Worker` on the `billing` queue dispatching `generate-invoice` and `attempt-payment` by job name (anything else throws); `:34-40` logs completions and failures; `:42-46` polls the outbox every 5 s; `:48-66` upserts the `billing-worker` heartbeat every 10 s.
- `jobs.ts:4-105` `generateInvoice`, one transaction: subscription and first active price (`:6-19`), period check (`:21-34`), coupon and tax (`:36-44`), calculation (`:46-57`), numbering and insert (`:59-83`), audit (`:85-93`), outbox (`:95-101`).
- `jobs.ts:107-166` `attemptPayment`, one transaction: invoice scoped to the organization (`:109-114`), skip if `paid` (`:116-118`), mock decision `total % 7 !== 0` (`:120`), payment row (`:121-134`), invoice `paid` on success (`:136-144`), audit and outbox (`:146-162`).
- `outbox-processor.ts:3-45` reads 10 `pending` events and marks each `processing`, then `processed`.

## Look for

- **The period check is not a lock.** Two `generate-invoice` jobs for the same subscription can both run the `findFirst` at `jobs.ts:21` before either inserts, and both create invoices. A unique constraint on `(subscriptionId, period start)`, or `SELECT ... FOR UPDATE` on the subscription row, makes the guard hold under concurrency.
- **No job options.** `billingQueue.add` (`apps/api/src/modules/billing.routes.ts:182`, `:243`) passes no `attempts`, `backoff` or `jobId`, so BullMQ's defaults apply: a failed job is not retried, and the same work can be enqueued twice. A deterministic `jobId` (for example `invoice:<subscriptionId>:<periodStart>`) makes enqueueing itself idempotent.
- **Failed payments leave the invoice `open`.** Nothing moves it toward `uncollectible`, and the subscription never becomes `past_due`, although both states exist.
- **Worker audit rows have no actor** (`actorUserId` is omitted), and use different action names from the API (`worker.invoice.generated` versus `billing.invoice.generated`).
- **The outbox "processing" state is a trap.** If the process dies between the two updates (`outbox-processor.ts:14-30`), the event stays `processing` forever. The `catch` block only covers the logging and the second update, and it rethrows, so one failure also skips the rest of that batch until the next tick.

## Exercises

1. **Goal:** with the worker stopped, create a subscription in the UI, then start the worker. **Check:** the queued `generate-invoice` job runs on startup and the invoice appears — BullMQ kept the job in Redis while no worker was running.
2. **Goal:** enqueue `generate-invoice` twice for the same subscription with the same `jobId`. **Check:** BullMQ keeps one job, and only one invoice is created.
3. **Goal:** make a payment fail on purpose: create a plan priced so the total is a multiple of 7 (for example a price of 700 cents, 1 seat, no coupon and no tax). **Check:** the payment row is `failed` with `failureCode` `card_declined`, and the invoice stays `open`.
4. **Goal:** reclaim stuck outbox events. `OutboxEvent` has no timestamp for when processing started, so first add a nullable `lockedAt` column (see `docs/migration-discipline.md`, expand and contract), set it when marking `processing`, and also select `processing` rows whose `lockedAt` is older than a minute. **Check:** an event you set to `processing` with a `lockedAt` ten minutes ago in Prisma Studio is processed on the next tick.
