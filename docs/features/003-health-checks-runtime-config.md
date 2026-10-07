# Feature 003: Health Checks And Runtime Configuration

Backlog ID: `BIL-003`

## Goal

Give the API and worker clearer startup and operational behavior.

## Impacted Areas Of The Codebase

- `apps/api/src/config.ts`
- `apps/api/src/app.ts`
- `apps/worker/src/index.ts`
- `packages/db/prisma/schema.prisma`

## Implementation Approach

Validate runtime environment variables with Zod, add `/health/live` and `/health/ready`, and let the worker write a heartbeat row that the API can report.

## Design Considerations

Liveness means the process can answer. Readiness means important dependencies are reachable. These are different checks because a process can be alive but unable to safely serve traffic.

## Build Journal

Added config validation for API and worker startup, database readiness checks, and a `WorkerHeartbeat` model. The worker periodically upserts its heartbeat and the API reports whether that heartbeat is fresh.

A junior developer should notice that this feature crosses process boundaries without inventing a new service. The tradeoff is that readiness is still simple: Redis queue health is implied by worker behavior rather than measured directly.

## Verified against this snapshot

- API config: `apps/api/src/config.ts:3-9` validates `API_PORT` (positive integer, default 4000), `REDIS_URL` (URL, default `redis://localhost:6379`) and `DEFAULT_USER_EMAIL` (email, default `owner@acme.test`) when the module is imported. `DATABASE_URL` is **not** in the schema; Prisma reads it on its own and fails at the first query if it is missing.
- Worker config: `apps/worker/src/index.ts:8-12` validates only `REDIS_URL`.
- Health: `/health/live` (`apps/api/src/app.ts:22-24`) and `/health/ready` (`:26-50`). Readiness returns 200 with `worker: "unknown"` when the heartbeat is stale or missing, and 503 only if the database query fails.
- Heartbeat: `WorkerHeartbeat` model (`packages/db/prisma/schema.prisma:334`), upserted every 10 s by the worker (`index.ts:48-66`), considered fresh for 30 s by the API (`app.ts:38`).

## Look for

- **`DEFAULT_USER_EMAIL` is a security setting disguised as config.** It decides who an anonymous request is (`apps/api/src/shared/auth.ts:15`). A production config schema should not give it a default.
- **Readiness never fails on the worker or Redis.** A load balancer would keep sending traffic to an API whose subscription endpoint fails on `billingQueue.add`.
- **The heartbeat stays `running` after a crash**; it just stops being updated. The API's 30-second freshness check is what turns that into `unknown`, and nothing records why the worker stopped.

## Exercises

1. **Goal:** start the API with `API_PORT=abc`. **Check:** it exits at startup with a Zod error naming `API_PORT`, before printing "Billing API listening".
2. **Goal:** add `DATABASE_URL` to the API config schema (`z.string().url()`). **Check:** starting the API without it fails immediately with a Zod error instead of on the first request.
3. **Goal:** with API and worker running, call `/health/ready`, stop the worker, wait 35 seconds and call it again. **Check:** `dependencies.worker` changes from `ready` to `unknown` while the HTTP status stays 200; then decide whether it should be 503 and write down why.
