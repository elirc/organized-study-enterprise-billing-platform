# Feature 002: Local Dev, CI, And Quality Gates

Backlog ID: `BIL-002`

## Goal

Make the project easy to verify locally and in CI with the same commands.

## Impacted Areas Of The Codebase

- Root `package.json` scripts.
- `.github/workflows/ci.yml`.
- Prisma client generation as a required CI step.

## Implementation Approach

Add a root `verify` script that runs typecheck, tests, and build. Add a GitHub Actions workflow that installs dependencies, generates Prisma client code, and runs verification against PostgreSQL and Redis service containers.

## Design Considerations

The workflow intentionally runs boring checks first. A junior developer should learn that CI is not magic; it is a repeatable version of local verification with clean dependencies.

## Build Journal

Implemented a root `npm run verify` command and CI workflow. The workflow uses Node 22, npm cache, PostgreSQL 16, and Redis 7. It runs `npm ci`, `npm run db:generate`, and `npm run verify`.

This connects to the rest of the system by making generated Prisma client code and workspace builds explicit. The tradeoff is that CI does not yet run browser E2E tests or apply real migrations; those arrive later when Playwright and migration discipline mature.

## Verified against this snapshot

- `npm run verify` (root `package.json`) = `typecheck && test && build`. `typecheck` first builds `@billing/domain` and `@billing/db` (the apps import their compiled `dist/` output), then runs every workspace's `typecheck` script.
- `.github/workflows/ci.yml` matches the description: Node 22 with npm cache, PostgreSQL 16 and Redis 7 service containers with health checks, then `npm ci`, `npm run db:generate`, `npm run verify`. Whether these runs passed is the author's record; the snapshot has no CI history.
- What the gate actually tests: the five tests in `packages/domain/src/domain.test.ts`. `@billing/api` runs `vitest run --passWithNoTests` with no test files; `@billing/web`, `@billing/worker` and `@billing/db` have no `test` script. The database and Redis containers are started, but no step uses them.
- The backlog entry (`docs/backlog.md`, BIL-002) also asks for linting, formatting, worker tests and a failure-triage guide. None of those exist yet; there is no ESLint or Prettier configuration in the repo.

## Exercises

1. **Goal:** add one API test that needs no database, for example `GET /health/live` through `supertest` against `createApp()`. **Check:** `npm test` reports the new test in `@billing/api`. (Importing `createApp` also imports the BullMQ queue, which connects to Redis at import time, so either start Redis or mock `shared/queues.ts`.)
2. **Goal:** make CI use its PostgreSQL container: add `npm run db:push` and `npm run db:seed` before `verify`, and an API test that reads `/api/identity/me`. **Check:** the test sees `owner@acme.test` with role `owner`.
3. **Goal:** add ESLint to `packages/domain` with `no-restricted-globals` (or `no-restricted-syntax`) forbidding `parseFloat`. **Check:** introducing `parseFloat` in `invoice-calculator.ts` makes `npm run verify` fail.
