# Architecture Journal

This journal records what changed, why it changed, and what a growing engineer should notice. It is deliberately written as a learning artifact, not a changelog.

## Entry 001: Initial System Shape

The first decision was to build a modular monolith instead of separate services. Billing systems have many workflows that must stay transactionally consistent: subscription creation, invoice generation, payment attempts, webhooks, retries, audit logs, and customer-visible state. Keeping these in one backend makes the first version easier to reason about while still allowing clear module boundaries.

The repository is split into apps and packages. Apps are deployable processes or user interfaces. Packages contain shared code with no direct HTTP concerns. This is a common enterprise pattern because it lets teams keep domain rules reusable without making every concept a network call.

The key teaching idea is that "monolith" does not mean "one big file." A good monolith has strict internal boundaries, boring deployment, and visible workflows.

## Entry 002: Verification First Backlog Batch

After the initial build, the first backlog batch added quality gates, CI, health checks, runtime config validation, worker heartbeat tracking, and migration discipline documentation.

This was deliberately chosen before deeper product features. Enterprise systems become hard to learn when every change feels risky. A `verify` command, CI workflow, and health endpoints create a safety loop: change something, run checks, inspect failures, fix the real cause, and repeat.

The sub-agent was useful for drafting the broad backlog while the main implementation stayed focused on code. The upside was parallel progress on planning and implementation. The downside is coordination: delegated work needs a narrow write area and a clear final artifact, otherwise two agents can easily step on the same files or produce plans that do not match the code. For this project, docs-only delegation worked well.

## Entry 003: Snapshot Review (2026-10 docs pass)

Entries 001 and 002 are the author's record; the repository is a single-commit snapshot, so the order of work and the sub-agent experience cannot be re-checked. This entry records what the code shows about each decision today.

### The modular monolith holds, but the invoice logic is duplicated

The apps/packages split is real: `packages/domain` has no database or HTTP imports, and both `apps/api` and `apps/worker` import `calculateInvoice` from it. But the code *around* the calculation — price lookup, coupon and tax selection, invoice numbering, audit and outbox writes — exists twice: `createInvoiceForSubscription` in `apps/api/src/modules/billing.routes.ts:252-341` and `generateInvoice` in `apps/worker/src/jobs.ts:4-105`. They have already drifted: only the worker checks for an existing invoice in the period (`jobs.ts:21-34`), and the two write different audit actions (`billing.invoice.generated` versus `worker.invoice.generated`). `system-design-current.md` lists this duplication as a tradeoff; the drift is the cost.

### "Verification first" covers the domain package only

The `verify` script (root `package.json`) runs typecheck, tests and build, and CI runs it against PostgreSQL and Redis service containers (`.github/workflows/ci.yml`). But the only test file is `packages/domain/src/domain.test.ts` (five tests), `apps/api` passes with no tests (`--passWithNoTests`), and CI never pushes the schema or migrates, so the database containers are started but unused by any test. The safety loop exists; it guards the money math, not the workflows.

### Tenant context is explicit, authentication is not

`attachTenantContext` (`apps/api/src/shared/auth.ts:7-45`) resolves organization, user and role from the database on every request, and every billing query adds `organizationId`. Two exceptions break the pattern: the mock webhook updates a payment by ID alone (`payments.routes.ts:51-59`), and `GET /api/payments/events` includes events with no organization. And because missing headers default to the owner, "development header auth" is effectively "everyone is the owner".

### Outbox and idempotency are shapes, not guarantees

Outbox rows are written in the same transaction as the business change — the important half of the pattern. The processor (`apps/worker/src/outbox-processor.ts`) only logs events and has no claim step. `runIdempotent` stores the key after the work, so it deduplicates sequential replays but not concurrent ones.

### What to decide next

1. **One invoice service.** Move invoice creation into a shared module (for example in `packages/domain` plus a thin persistence function in `packages/db`) used by both the route and the worker.
2. **Where status changes happen.** `packages/domain/src/billing-state.ts` defines the transitions, but nothing outside the tests calls them. Decide that every status write goes through `transitionInvoice`/`transitionSubscription`.
3. **API tests before BIL-010.** Authentication sessions will rewrite `attachTenantContext`; tenant-isolation tests written first would tell you whether the rewrite kept the guarantees.

## Exercise

**Goal:** write Entry 004 for decision 1: the options (shared function in a package; the API calls the worker through the queue only; the worker calls an API endpoint), the choice, and why.

**Check:** after implementing it, `grep -rn "INV-" apps packages` finds the numbering logic in exactly one place, and both the manual endpoint and the worker job produce the same audit action.
