# Code Walkthrough Index

Read these files first when learning the codebase. They cover the architectural center of gravity rather than every implementation detail.

1. [Prisma Schema](prisma-schema.md)
2. [Domain Rules](domain-rules.md)
3. [API App Composition](api-app.md)
4. [Billing Routes And Workflow](billing-routes.md)
5. [Worker Jobs](worker-jobs.md)
6. [Frontend App Surface](frontend-app.md)

The suggested reading style is workflow-first. Pick an action, such as creating a subscription, then trace how data moves through frontend, API, domain helpers, database records, outbox events, and worker jobs.

## What each walkthrough adds

Every walkthrough now ends with three parts: **Line by line** (where each important section sits in this snapshot), **Look for** (review findings — bugs, races and gaps verified by reading the code) and **Exercises** (each with a goal and an observable check). A good order for a first pass:

1. [Prisma Schema](prisma-schema.md) — the 21 models, which are tenant-owned, and which constraints are missing.
2. [Domain Rules](domain-rules.md) — the money math, and the state machines nobody calls.
3. [API App Composition](api-app.md) — middleware order and why every error is a 400.
4. [Billing Routes And Workflow](billing-routes.md) — the route table and the invoice duplication.
5. [Worker Jobs](worker-jobs.md) — retries, the period check, and the outbox.
6. [Frontend App Surface](frontend-app.md) — the idempotency key that changes on every click.

Then read `docs/system-design-current.md`, which strings the same files together as end-to-end workflows.
