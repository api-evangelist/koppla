---
name: Report construction site progress to koppla
description: Record status reports against scheduled orders, update task status,
  and attach jobsite photos — the write path a field or site-progress integration
  uses.
api: graphql/koppla-graphql.graphql
endpoint: https://api.koppla.de/api/graphql/v1
operations:
  - Query.projectOrders
  - Query.projectOrderStatusReports
  - Mutation.createOrderStatusReports
  - Mutation.updateOrderTaskStatus
  - Mutation.createOrderPhotoV2
  - Query.projectPhotos
generated: '2026-07-19'
method: generated
source: graphql/koppla-graphql-introspection.json
---

# Report construction site progress to koppla

This is the write path: telling koppla what actually happened on site against
the planned schedule. It mutates customer project data — read the safety notes.

> koppla's API is Enterprise-contract only and undocumented publicly. These steps
> are grounded in the live GraphQL schema, not in a provider guide.

## Safety first — no idempotency

**koppla's API has no idempotency mechanism.** There is no idempotency-key
header or argument anywhere in the schema. Mutations accept Relay's
`clientMutationId`, but that is a correlation id echoed back to you — it does
**not** deduplicate retries.

Consequences you must design for:

- A retried `createOrderStatusReports` can create a **duplicate** report.
- On a network timeout, do **not** blindly retry. Re-query
  `Query.projectOrderStatusReports(project:)` to see whether the write landed,
  then retry only if it did not.
- Keep your own client-side dedupe key and check before resubmitting.

## Step 1 — locate the order

Find the scheduled activity you are reporting against (see the
`koppla-browse-project-schedule` skill):

```graphql
query($project: ID!) {
  projectOrders(project: $project, first: 100,
                inPeriodFrom: "2026-07-01", inPeriodTo: "2026-07-31") {
    edges { node { id name status progress } } } }
```

## Step 2 — file the status report

Use the **plural** `createOrderStatusReports` — the singular
`createOrderStatusReport` is deprecated in favour of it. All koppla mutations
take a single `input` argument.

```graphql
mutation($input: <see schema>) {
  createOrderStatusReports(input: $input) { clientMutationId } }
```

Resolve the exact input shape from `graphql/koppla-graphql.graphql` before
calling — the schema is the contract, and this repo carries no invented fields.
Batching several reports into one call is both cheaper and safer than looping,
precisely because retries are not idempotent.

## Step 3 — task status and photos

- `Mutation.updateOrderTaskStatus(input:)` — move an individual task's status.
- `Mutation.createOrderPhotoV2(input:)` — attach a jobsite photo. Use the
  **V2** mutation: `createOrderPhoto` is deprecated and carries an upload size
  restriction that V2 removes. Uploads use the `Upload` scalar, so send a
  GraphQL multipart request rather than plain JSON.
- Verify with `Query.projectPhotos(filter:)`.

## Verify, then stop

After a write, re-read `Query.projectOrderStatusReports(project:)` (newest
first) and confirm your report is present exactly once. If you find duplicates
from an earlier retry, surface that to a human — this skill does not delete
data.

## Errors

Responses are HTTP 200 with an `errors[]` array; a partial `data` payload can
accompany failures, so check both. `extensions.code = UNAUTHENTICATED` means the
token is missing or invalid. See `errors/koppla-error-codes.yml` and
`conventions/koppla-conventions.yml`.
