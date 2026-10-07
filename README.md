# Enterprise Billing Platform

A learning-oriented multi-tenant SaaS billing and subscription platform. The project is intentionally shaped like a realistic modular monolith: API, frontend, database package, background worker, shared domain rules, tests, and architecture documentation.

## Quick Start

```bash
npm install
cp .env.example .env
docker compose up -d
npm run db:generate
npm run db:push
npm run db:seed
npm run dev:api
npm run dev:web
```

Run `npm run dev:worker` in a third terminal: without it, subscriptions never get their automatic first invoice and "queue payment" never produces a payment.

**There is no `.env.example` in this snapshot**, so the `cp` step fails. Create `.env` yourself. The values below match `docker-compose.yml` and the CI workflow (`.github/workflows/ci.yml`):

```env
DATABASE_URL=postgresql://billing:billing@localhost:5432/billing_platform?schema=public
REDIS_URL=redis://localhost:6379
```

`API_PORT` (default 4000) and `DEFAULT_USER_EMAIL` (default `owner@acme.test`) are optional (`apps/api/src/config.ts:3-7`); the web app reads `VITE_API_URL` (default `http://localhost:4000`, `apps/web/src/api.ts:1`). Prisma CLI commands run inside `packages/db` (they are workspace scripts), so check that Prisma finds your `DATABASE_URL` from there — exporting it in your shell is the simplest way to be sure.

## Workspace Layout

- `apps/api`: Express API and backend modules (`src/app.ts`, `src/modules/*.routes.ts`, `src/shared/*`).
- `apps/web`: React admin dashboard and customer portal (one `App.tsx`, one `api.ts`).
- `apps/worker`: BullMQ background worker (invoice and payment jobs, outbox polling, heartbeat).
- `packages/domain`: Shared domain rules and their tests (money, invoice calculation, permissions, state transitions, idempotency hashing).
- `packages/db`: Prisma schema, Prisma client, and seed data.
- `docs`: Architecture, feature plans, build journals, walkthroughs, and backlog.

## The API at a glance

All routes are mounted in `apps/api/src/app.ts:52-56`. Every request except `/health*` goes through `attachTenantContext` first.

| Method | Path | Permission |
|---|---|---|
| GET | `/health`, `/health/live`, `/health/ready` | none |
| GET | `/api/identity/me` | any member |
| GET | `/api/catalog/plans`, `/api/catalog/coupons` | any member |
| POST | `/api/catalog/plans` | `catalog:manage` |
| GET | `/api/billing/dashboard`, `/customers`, `/subscriptions`, `/invoices` | `billing:read` |
| POST | `/api/billing/customers`, `/subscriptions`, `/subscriptions/:id/invoices` | `billing:manage` |
| POST | `/api/billing/invoices/:id/pay` | `payments:manage` |
| GET | `/api/payments/events` | `payments:manage` |
| POST | `/api/payments/webhooks/mock` | any member |
| GET | `/api/platform/audit-logs`, `/api/platform/outbox` | `audit:read` |
| GET | `/api/platform/portal` | any member |

Permissions per role are in `packages/domain/src/permissions.ts:14-38`.

## Who you are

There is no login. `attachTenantContext` (`apps/api/src/shared/auth.ts:7-45`) reads `x-org-slug` and `x-user-email`, and **falls back to `acme` and `owner@acme.test` when they are missing**, so an anonymous request is the organization owner. The web app always sends exactly those two values (`apps/web/src/api.ts:33-38`). The seed creates one organization (`acme`) with one user, the owner, so the RBAC table above cannot be exercised with seed data alone.

## Known gaps (verified in this snapshot)

- **Cross-tenant webhook write.** `POST /api/payments/webhooks/mock` updates the payment by `id` alone (`apps/api/src/modules/payments.routes.ts:51-59`), with no organization filter, no signature and no permission beyond membership. Any member of any organization who knows a payment ID can mark it succeeded, which also marks that payment's invoice `paid`.
- **Every error is a 400 or 403.** The error handler (`apps/api/src/app.ts:58-62`) returns 403 if the message contains "Missing permission" and 400 for everything else — including not-found records, database outages and programming errors.
- **Idempotency is check-then-act.** `runIdempotent` (`apps/api/src/shared/idempotent.ts`) records the key only after the work succeeds, so two concurrent requests with the same key both create a subscription; and the web app generates a fresh key on every click (`apps/web/src/App.tsx:103`).
- **Duplicate invoices.** The worker skips a subscription that already has an invoice in its period (`apps/worker/src/jobs.ts:21-34`); the manual endpoint `POST /api/billing/subscriptions/:id/invoices` has no such check. Invoice numbers come from `count + 1` (`billing.routes.ts:294-300`), which collides under concurrency against the `@@unique([organizationId, number])` index.
- **The newest active coupon applies to every invoice** in the organization (`billing.routes.ts:274-277`, `jobs.ts:37-40`); coupons are not attached to customers or subscriptions.
- **The state machines are not wired in.** `transitionSubscription`/`transitionInvoice` (`packages/domain/src/billing-state.ts`) are only called from tests; routes and jobs write statuses directly.
- **Only the domain package has tests.** `apps/api` runs `vitest run --passWithNoTests` and has no test files; the worker and web apps have no test script.

## Where to read next

Start with [docs/architecture-planning.md](docs/architecture-planning.md), then read [docs/architecture-journal.md](docs/architecture-journal.md). For code, follow [docs/walkthroughs/README.md](docs/walkthroughs/README.md); each walkthrough now ends with line references, review notes and exercises. [docs/system-design-current.md](docs/system-design-current.md) traces the main workflows step by step.

## About this snapshot

This repository is a single-commit import. Statements in the docs about how the project was built (sub-agent use, CI runs, build results) are the author's record and cannot be re-checked from the snapshot. Claims about code were re-checked against this snapshot during a 2026-10 docs pass.
