# Migration Discipline

Database changes should be treated as application behavior, not plumbing.

## Local Workflow

1. Edit `packages/db/prisma/schema.prisma`.
2. Run `npm run db:generate`.
3. For early local-only experiments, run `npm run db:push`.
4. For durable feature work, run `npm run prisma:migrate --workspace @billing/db`.
5. Update seed data only when the new schema needs realistic local examples.
6. Run `npm run verify`.

## Review Rules

- Every tenant-owned table needs an `organizationId` or a clear reason why it does not.
- Money columns must be integer cents plus currency.
- State changes should prefer explicit enums over boolean combinations.
- Avoid destructive migrations unless there is a documented backup and rollback path.
- Avoid enum changes without a transition plan; production databases can make enum edits awkward.
- Add indexes for common tenant-scoped reads and workflow queues.
- Keep audit logs append-only.
- Do not hide cross-tenant data leaks in seed data.

## Expand And Contract

Prefer safe multi-step migrations for risky changes:

1. Expand: add nullable column or new table.
2. Backfill: copy or compute existing data.
3. Read: update application code to read from the new shape.
4. Write: update application code to write the new shape.
5. Contract: remove old column only after the system no longer depends on it.

## Deployment

Production deployment should use:

```bash
npm run prisma:deploy --workspace @billing/db
```

`db push` is for local learning speed, not production discipline.

## Where this repository stands (checked against this snapshot)

- **There are no migrations yet.** `packages/db/prisma/` contains only `schema.prisma` and `seed.ts`; there is no `migrations/` folder. The `prisma:migrate` and `prisma:deploy` scripts exist (`packages/db/package.json`), but `prisma migrate deploy` would have nothing to apply. Every database built from this repo so far was built with `db push`.
- **CI does not touch the schema.** `.github/workflows/ci.yml` runs `npm ci`, `npm run db:generate` and `npm run verify`. Generating the client checks that the schema parses; nothing checks that it can be applied, or that a migration matches it.
- **The first migration is a baseline.** When you create it, any database that was built with `db push` already has these tables. Such databases need the baseline marked as applied (`prisma migrate resolve --applied <name>`) rather than run.

### The review rules, applied to today's schema

| Rule | Status in `packages/db/prisma/schema.prisma` |
|---|---|
| Tenant-owned tables carry `organizationId` | Mostly. `Price`, `Entitlement`, `InvoiceLine` and `TeamMember` do not; they hang off a parent that does (`Plan`, `Invoice`, `Team`). `WebhookEvent.organizationId` and `IdempotencyKey.organizationId` are **nullable**. |
| Money is integer cents plus currency | Yes for `Price`, `Invoice`, `Payment`, `InvoiceLine`. `currency` is a free `String`, while the domain only knows `USD`, `EUR`, `GBP` (`packages/domain/src/money.ts:1`). |
| Explicit enums for state | Yes: `SubscriptionStatus`, `InvoiceStatus`, `PaymentStatus`, `OutboxStatus`, `OrganizationStatus`, `Role`. `WorkerHeartbeat.status` is a free string. |
| Indexes for tenant-scoped reads and queues | `Subscription(organizationId, status)`, `Invoice(organizationId, status)`, `AuditLog(organizationId, createdAt)`, `OutboxEvent(status, availableAt)`. `Customer` lists order by `createdAt` with no matching index. |
| Audit logs append-only | By convention only: no code updates or deletes `AuditLog`, but nothing prevents it. |
| No cross-tenant leaks in seed data | The seed has one organization, so it cannot leak — and cannot demonstrate isolation either. |

One rule the list does not mention: **a nullable column in a unique index.** `IdempotencyKey` is unique on `(organizationId, key)` with `organizationId` nullable. In PostgreSQL, NULLs are distinct in unique indexes by default, so two rows with a NULL organization and the same key are both allowed. The API always passes an organization today, so this is latent.

## Exercises

Each has a **Goal** and a **Check** you can observe.

1. **Create the baseline.**
   *Goal:* against an empty database, run `npm run prisma:migrate --workspace @billing/db -- --name init`.
   *Check:* `packages/db/prisma/migrations/<timestamp>_init/migration.sql` exists and contains a `CREATE TABLE` for every model (21 tables plus the enum types), and `npm run prisma:deploy --workspace @billing/db` against a second empty database applies it cleanly.
2. **Practise expand and contract.**
   *Goal:* attach coupons to subscriptions (today the newest active coupon applies to every invoice). Write the five steps for adding a nullable `Subscription.couponId`, backfilling it, switching the invoice code to read it, and only then removing the organization-wide lookup.
   *Check:* each step is a separate migration or code change that can be deployed alone, and after step 3 an invoice for a subscription without a coupon has `discount` 0 even while `WELCOME10` is active.
3. **Make CI apply the schema.**
   *Goal:* add a `prisma migrate deploy` step to the CI workflow after `db:generate`.
   *Check:* a pull request that edits `schema.prisma` without adding a migration still passes `migrate deploy` — so also add `prisma migrate diff --exit-code` (from the migrations folder to the schema) and confirm that such a pull request now fails.
