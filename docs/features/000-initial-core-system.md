# Feature 000: Initial Core System

## Goal

Create the first production-shaped version of the SaaS billing platform: tenant-aware users, plans, customers, subscriptions, invoices, mock payments, audit logs, worker jobs, and a frontend dashboard.

## Impacted Areas Of The Codebase

- `packages/domain`: money helpers, permissions, billing state transitions, invoice calculation.
- `packages/db`: Prisma schema and seed data.
- `apps/api`: HTTP routes, authorization middleware, billing workflows, audit/outbox writes.
- `apps/worker`: background invoice and payment processing.
- `apps/web`: admin dashboard and customer portal screens.
- `docs`: architecture plan, journal, walkthroughs, and backlog.

## Implementation Approach

1. Scaffold npm workspaces with shared TypeScript configuration.
2. Define Prisma models around tenant ownership and billing workflows.
3. Implement domain helpers first so API and worker code share business rules.
4. Build API modules around identity, catalog, billing, payments, and platform operations.
5. Build worker job handlers for invoice finalization and mock payment attempts.
6. Build frontend screens that exercise real API flows.
7. Add tests around the highest-risk domain logic.

## Design Considerations

- Tenant isolation must be enforced in backend queries, not just UI filters.
- Money must be integer-based and never use floating-point arithmetic.
- Subscription and invoice state transitions should be explicit.
- Webhook/idempotency paths should be shaped early because retrofitting them is painful.
- The UI should look like an operational admin tool: compact, scannable, and workflow-oriented.

## Build Journal

Implemented the initial modular monolith:

- Npm workspaces with `apps/api`, `apps/web`, `apps/worker`, `packages/domain`, and `packages/db`.
- Prisma schema for tenants, users, memberships, teams, plans, prices, entitlements, customers, subscriptions, invoices, payments, webhooks, idempotency keys, audit logs, outbox events, preferences, and worker heartbeats.
- Express API modules for identity, catalog, billing, payments, and platform inspection.
- BullMQ worker for invoice generation, payment attempts, outbox processing, and heartbeat reporting.
- React frontend with operational screens for dashboard, customers, subscriptions, catalog, customer portal, and audit trail.
- Domain tests for money handling, invoice calculation, permissions, workflow transitions, and idempotency hashing.

The most important lesson is that business workflows cross layers. Creating a subscription touches authorization, tenant scoping, database writes, audit logs, outbox events, worker jobs, and frontend refresh behavior. A mid-level engineer learns to trace that whole path instead of only reading the route handler.

Tradeoffs:

- Authentication is development-header based for now; real sessions are a planned backlog feature.
- The payment provider is deterministic and mocked so state transitions are easy to inspect.
- API integration tests and Playwright E2E tests are intentionally left for later backlog work after the core flow is stable.

## Verified against this snapshot

The build journal above is the author's record. Checked against the code:

| Journal claim | What the code shows |
|---|---|
| Npm workspaces with three apps and two packages | Yes: root `package.json` `workspaces: ["packages/*", "apps/*"]`; package names `@billing/api`, `@billing/web`, `@billing/worker`, `@billing/db`, `@billing/domain`. |
| Prisma schema for the listed entities | Yes, 21 models (`packages/db/prisma/schema.prisma`). |
| API modules for identity, catalog, billing, payments, platform | Yes, `apps/api/src/modules/*.routes.ts`, mounted in `app.ts:52-56`. |
| Worker for invoices, payments, outbox, heartbeat | Yes, `apps/worker/src/index.ts`, `jobs.ts`, `outbox-processor.ts`. The outbox processor only logs events. |
| React screens for dashboard, customers, subscriptions, catalog, portal, audit | Yes, `apps/web/src/App.tsx:19` and components from `:211`. |
| Domain tests for money, invoices, permissions, transitions, idempotency | Yes, one test each in `packages/domain/src/domain.test.ts`. |

Design considerations versus code:

- *"Tenant isolation must be enforced in backend queries."* Done in every billing, catalog and platform query, with one exception: the mock webhook updates a payment by ID without `organizationId` (`apps/api/src/modules/payments.routes.ts:51-59`).
- *"Subscription and invoice state transitions should be explicit."* Explicit enums exist and `packages/domain/src/billing-state.ts` defines the transitions, but no route or job calls them.
- *"Webhook/idempotency paths should be shaped early."* Shaped: `WebhookEvent` deduplicates by `providerEventId`, and `runIdempotent` stores responses per key. Neither is safe under concurrent requests yet (see `docs/system-design-current.md`).

## Exercise

**Goal:** take the journal's "most important lesson" (creating a subscription touches authorization, tenant scoping, writes, audit, outbox, jobs and the frontend refresh) and list the file and line for each of those seven steps.

**Check:** your list includes `apps/api/src/shared/auth.ts:55-62`, `billing.routes.ts:137-145`, `:150-162`, `:164-171`, `:173-177`, `:182-185`, and the `await refresh()` at the end of `createSubscription` in `apps/web/src/App.tsx` (line 105).
