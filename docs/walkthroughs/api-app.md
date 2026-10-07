# Walkthrough: `apps/api/src/app.ts`

## What The File Does

`app.ts` composes the Express application: middleware, health checks, module routers, and shared error handling.

## Why It Exists

This file is the API entry point for request flow. It should stay small because business logic belongs in modules and shared services, not app setup.

## How It Connects

- `server.ts` starts the app.
- `attachTenantContext` loads the authenticated tenant context.
- Module routers under `apps/api/src/modules` own feature-specific endpoints.
- Health checks use Prisma to report readiness.

## Important Sections

- JSON and CORS middleware.
- `/health/live` for process liveness.
- `/health/ready` for database readiness and worker heartbeat visibility.
- Router mounting for identity, catalog, billing, payments, and platform modules.

## Senior-Level Concepts

The health endpoints skip tenant auth because deployment platforms and load balancers need to call them. Readiness includes dependency checks; liveness does not.

## How To Read It

Read top to bottom once, then jump into the mounted router for the workflow you care about.

## Line by line (this snapshot)

- `:14-16` CORS (any origin), JSON body parsing, then `attachTenantContext` for **every** request. The health routes are skipped inside the middleware (`shared/auth.ts:9-12`), not by registration order.
- `:18-20` `/health` and `:22-24` `/health/live` return static JSON.
- `:26-50` `/health/ready`: `SELECT 1`, then the `billing-worker` heartbeat. Worker state is reported as `ready` (heartbeat under 30 s old) or `unknown`; the HTTP status is 200 either way, and 503 only when the database query throws.
- `:52-56` five routers under `/api/identity`, `/api/catalog`, `/api/billing`, `/api/payments`, `/api/platform`.
- `:58-62` the only error handler: 403 if the message contains "Missing permission", otherwise **400**.

`server.ts` imports `config` (`config.ts`), which parses `process.env` with Zod at import time (`config.ts:9`), so an invalid `REDIS_URL` or `API_PORT` stops the API before it listens.

## Look for

- **Status codes come from message text.** A Prisma "record not found", a unique-constraint violation, a lost Redis connection and a bug all become 400 with the raw error message in the body — which can include Prisma internals. Typed errors (`NotFoundError`, `ConflictError`) mapped to 404/409/500 would separate client mistakes from server faults.
- **Readiness does not check Redis**, although subscription creation and payment queueing need it (`shared/queues.ts`).
- **Importing `billing.routes.ts` opens a Redis connection** (`shared/queues.ts:5-9` runs at import), so any test that imports `createApp` needs Redis or a mock.

## Exercises

1. **Goal:** call `GET /api/billing/invoices` with `x-user-email: nobody@acme.test`. **Check:** the response is 400 with "No membership found for nobody@acme.test in organization acme. Run db:seed first." — then change the middleware to answer 401 and confirm the status changes.
2. **Goal:** add a `NotFoundError` class, throw it where a `findFirstOrThrow` would fail, and map it to 404 in the handler. **Check:** `POST /api/billing/invoices/does-not-exist/pay` returns 404 instead of 400.
3. **Goal:** add a Redis `PING` to `/health/ready`. **Check:** with `docker compose stop redis`, readiness returns 503 and names Redis; with Redis back, it returns 200.
