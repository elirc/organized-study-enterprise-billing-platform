# Walkthrough: `apps/web/src/App.tsx`

## What The File Does

This file provides the initial React operator interface: dashboard, customers, subscriptions, catalog, customer portal, and audit views.

## Why It Exists

The frontend is intentionally workflow-shaped. It is not a marketing page; it is an operational surface for exercising backend workflows.

## How It Connects

- `api.ts` sends requests with development tenant headers.
- Dashboard and list views read API data.
- Action buttons create customers, create subscriptions, generate invoices, and queue payments.

## Important Sections

- `refresh`: loads the main data slices in parallel.
- Topbar actions: trigger core write workflows.
- View components: keep each operational screen readable while sharing one file for the initial version.

## Senior-Level Concepts

Frontend role checks are not security. This UI may hide or show actions later, but the backend remains the source of truth through `assertPermission`.

## How To Read It

Start at `App`, then follow one action button into `postJson`, then into the matching API route.

## Line by line (this snapshot)

- `App.tsx:30` the `App` component; `:19` the six views (`dashboard`, `customers`, `subscriptions`, `catalog`, `portal`, `audit`).
- `:43-69` `refresh` loads seven endpoints with `Promise.all`; if any one fails, the whole refresh fails and only an error string is shown.
- `:75-78` "active" records are simply the first item of each list: `plans[0]`, `customers[0]` (the newest customer), `subscriptions[0]`, and the first `open` invoice.
- `:80-86` `createCustomer` invents a name and a `customer-<timestamp>@example.test` email.
- `:88-106` `createSubscription` sends 3 seats, 30 days, trial on, with idempotency key `sub_<customer>_<plan>_<Date.now()>` (`:103`).
- `:108-116` `generateInvoice` and `:118-126` `queuePayment` call the manual invoice endpoint and the payment queue.
- `:128-145` view switch; components from `:211` (`Dashboard`) to `:356` (`InvoiceTable`).
- `api.ts:1` base URL from `VITE_API_URL`; `:33-38` the fixed development headers (`acme`, `owner@acme.test`).

## Look for

- **The idempotency key is useless from the UI.** It contains `Date.now()`, so two clicks produce two keys and two subscriptions. A key generated once per form submission (and reused on retry) is what makes idempotency work.
- **Write actions have no error handling.** `createCustomer`, `createSubscription`, `generateInvoice` and `queuePayment` `await postJson(...)` without `try/catch`; a 400 rejects the promise inside a click handler and the user sees nothing.
- **Queued work is not awaited.** `queuePayment` refreshes immediately after the 202, before the worker has run, so the invoice still shows `open` until the next manual refresh.
- **The portal is not a customer view.** `Portal` (`:322-332`) shows the newest customer's name above the organization's six newest invoices — not that customer's invoices.
- **Amounts are always formatted as USD** (`formatCents`, `:392-397`), even though prices can be EUR or GBP (`catalog.routes.ts:99`).

## Exercises

1. **Goal:** click "Create subscription" twice quickly. **Check:** the Subscriptions view shows two new subscriptions for the same customer. Change the key to be generated once per user intent (for example when the button is pressed, stored in state until the request succeeds) and confirm a retried request returns the same subscription.
2. **Goal:** wrap each write action in `try/catch` that calls `setError`. **Check:** with the API stopped, clicking "Create customer" shows an error message instead of doing nothing.
3. **Goal:** make the portal show only the selected customer's invoices (filter by `customerId`, or call `/api/platform/portal` and use its `invoices`). **Check:** after creating a second customer with no invoices, the portal for that customer shows an empty table.
