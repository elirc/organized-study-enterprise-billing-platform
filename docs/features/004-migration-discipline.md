# Feature 004: Migration Discipline

Backlog ID: `BIL-004`

## Goal

Document how schema changes should be made, reviewed, and verified.

## Impacted Areas Of The Codebase

- `packages/db/package.json`
- `docs/migration-discipline.md`

## Implementation Approach

Add Prisma migration scripts and write a migration discipline guide for local development, review, and deployment.

## Design Considerations

Database changes are product changes. Billing systems are especially sensitive because schema mistakes can corrupt money, state, or auditability.

## Build Journal

Added `prisma:migrate` and `prisma:deploy` scripts and a migration guide. The guide explains expand/contract migrations, enum caution, seed data expectations, and review rules.

The tradeoff is that the initial project still uses `db push` for fast local setup. Real production changes should move through generated migrations as the schema stabilizes.

## Verified against this snapshot

- `packages/db/package.json` has `prisma:migrate` (`prisma migrate dev`) and `prisma:deploy` (`prisma migrate deploy`), as described, alongside `prisma:push` and `seed`.
- `docs/migration-discipline.md` covers the local workflow, review rules, expand-and-contract and deployment.
- There is no `packages/db/prisma/migrations/` folder, so no migration has been generated yet, and CI does not apply the schema. The guide now has a "Where this repository stands" section that applies its own review rules to the current schema and lists exercises, starting with creating the baseline migration.

## Exercise

**Goal:** run the review rules in `docs/migration-discipline.md` against one schema change of your own, for example adding `Subscription.couponId` (today the newest active coupon applies organization-wide).

**Check:** your change adds an index for the new foreign key, keeps the column nullable until a backfill has run, and comes with a migration file plus an updated seed.
