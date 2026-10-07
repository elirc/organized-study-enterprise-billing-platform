# Walkthrough: `packages/domain/src`

## What These Files Do

The domain package contains business rules that should not live inside random HTTP handlers:

- `money.ts`: integer-cent money helpers.
- `invoice-calculator.ts`: line, discount, tax, and total calculation.
- `permissions.ts`: role-to-permission mapping.
- `billing-state.ts`: subscription and invoice state transitions.
- `idempotency.ts`: stable request hashing.

## Why They Exist

Shared domain helpers keep API, worker, and tests aligned. If invoice math changes, it should change in one place and be tested directly.

## How They Connect

- API routes import permissions and invoice calculation.
- Worker jobs reuse invoice calculation.
- Tests in `domain.test.ts` cover high-risk rules quickly without needing a database.

## Important Sections

Read `calculateInvoice` after `money.ts`. It shows a clear order: line subtotals, discount, tax, total. That order matters because billing disputes often come from hidden calculation rules.

## Senior-Level Concepts

Money is stored as integer cents, not floating point. Permissions are action-based, not screen-based. State transitions are explicit so invalid transitions fail near the source.

## How To Read It

Treat this package as the system's business vocabulary. Before changing a billing route, check whether the rule belongs here.

## Line by line (this snapshot)

- `money.ts:8-14` `money()` rejects non-integer cents; `:30-36` quantities must be non-negative integers; `:38-45` percentage discounts must be 0-100 and are rounded with `Math.round`; `:47-54` tax is basis points (825 = 8.25%), also rounded; `:16-28` `addMoney` refuses mixed currencies, and an empty list returns `0 USD` regardless of the invoice currency.
- `invoice-calculator.ts:28-49` the order is: line subtotals → sum → discount on the sum → tax on the discounted sum → total. The `discount` is derived as `subtotal - discountedSubtotal` (`:38`), so rounding happens once.
- `permissions.ts:14-38` five roles. Only `owner` has `org:manage` and `support:impersonate`, and no route checks either. `support` gets `audit:read` but not `payments:manage`.
- `billing-state.ts:12-26` transitions. `canceled`, `paid` and `void` are terminal; `uncollectible` can still become `paid` or `void`.
- `idempotency.ts:3-21` sorts object keys recursively before hashing, so `{a,b}` and `{b,a}` hash the same. Arrays keep their order.

## Look for

- **The state machines are not used.** No file outside `domain.test.ts` imports `transitionSubscription` or `transitionInvoice`; the API and worker write statuses directly (`jobs.ts:137-143`, `payments.routes.ts:54`, `:62-68`).
- **`requirePermission` is unused**; the API has its own `assertPermission` in `apps/api/src/shared/auth.ts` with a different error message, and the API's 403 mapping depends on that message ("Missing permission").
- **The tests are one example each.** `fast-check` and `effect` are root dev dependencies, but no property-based test uses them. Rounding edge cases (half cents), 0% and 100% coupons, and mixed currencies are untested.

## Exercises

1. **Goal:** add a property test with `fast-check`: for any integer unit price 0-100000, quantity 0-500, coupon 0-100 and tax 0-2500 basis points, `total === subtotal - discount + tax`. **Check:** the property holds for 1,000 runs (`fc.assert(..., { numRuns: 1000 })`).
2. **Goal:** call `calculateInvoice` with `couponPercentOff: 100`. **Check:** discount equals subtotal, tax is 0 and total is 0; write it as a test.
3. **Goal:** decide what `addMoney([])` should do for a EUR invoice with no lines, and change the signature so the currency is passed in. **Check:** a test for an empty EUR invoice returns `{ cents: 0, currency: "EUR" }`.
