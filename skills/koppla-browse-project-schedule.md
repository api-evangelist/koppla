---
name: Browse a koppla project schedule
description: List the construction projects you have access to, page through the
  orders (scheduled activities) of one project over a date window, and read its
  milestones and trade sequences.
api: graphql/koppla-graphql.graphql
endpoint: https://api.koppla.de/api/graphql/v1
operations:
  - Query.userProjects
  - Query.projects
  - Query.projectOrders
  - Query.order
  - Query.tradeSequences
  - Query.projectMilestones
generated: '2026-07-19'
method: generated
source: graphql/koppla-graphql-introspection.json
---

# Browse a koppla project schedule

koppla is construction scheduling software. A **project** is a construction
project; an **order** is a scheduled activity (a trade's work package) on that
project's schedule. This skill reads the schedule. It never writes.

> koppla's API is available under an Enterprise contract and is not publicly
> documented. You need a bearer token from koppla before any of this works.

## Before you start

- Endpoint: `POST https://api.koppla.de/api/graphql/v1`, content type `application/json`.
- Send `Authorization: Bearer <token>`. Without it, every data field returns a
  GraphQL error with `extensions.code = UNAUTHENTICATED` — the HTTP status is
  still 200, so **check `errors[]`, not the status code**.
- koppla is multi-tenant. `Query.projects` requires a `tenant: ID!` argument.
  If you do not know the tenant, start with `Query.userProjects`, which scopes to
  your own memberships and takes no tenant argument.

## Step 1 — find the project

Prefer `userProjects`; it needs no tenant id.

```graphql
query { userProjects(first: 25, orderBy: ["name"]) {
  pageInfo { hasNextPage endCursor }
  edges { node { id name } } } }
```

Use `Query.projects(tenant: $tenantId, ...)` instead when you are operating
across a whole tenant rather than your own memberships.

Do **not** use `Query.project` to fetch a single project — it is deprecated in
favour of `GET /projects/:projectId` on koppla's REST API.

## Step 2 — page through the schedule

Every list is a Relay cursor connection. Pass `first` plus `after`, and keep
going while `pageInfo.hasNextPage` is true, feeding `pageInfo.endCursor` back
as `after`. Never assume one page is the whole schedule.

```graphql
query($project: ID!, $after: String) {
  projectOrders(project: $project, first: 100, after: $after,
                inPeriodFrom: "2026-07-01", inPeriodTo: "2026-09-30") {
    pageInfo { hasNextPage endCursor }
    edges { node { id name startAt finishAt progress status } } } }
```

`inPeriodFrom` / `inPeriodTo` bound the window — always set them on a large
project rather than pulling the entire schedule.

Fetch one activity in detail with `Query.order(id:)`, or a known set with
`Query.orders(ids:)`.

## Step 3 — milestones and trade sequences

- `Query.tradeSequences(project:)` — the repeating trade sequence (takt) plan.
- `Query.projectMilestones(project:)` — milestones. **Deprecated**: koppla
  directs callers to `GET /projects/:projectId/milestones` on the REST API.
  Read it if you must, but plan the migration.

## Field selection and deprecation

Ask only for the fields you need — GraphQL selection sets are the sparse-fieldset
mechanism; there is no `expand` parameter.

45 fields in this schema carry `@deprecated` with a stated replacement. Before
selecting a field, check `graphql/koppla-graphql.graphql`. Common traps:

- `ProjectNode.status` → use `setupStatus`
- `OrderNode.subcontractor`, `ProjectNode.memberships` → contributor groups
- anything on `TicketNode` marked `TITO`
- `*V2` siblings (`workingTimesV2`, `exceptionsV2`) supersede the originals

## Errors

Check `errors[]` on every response. `extensions.code` is the machine-readable
signal; `UNAUTHENTICATED` is the one you will hit first. Gateway-level failures
come back as plain `{"message": "Missing Authentication Token"}` with HTTP 403
instead of a GraphQL envelope — handle both shapes. See
`errors/koppla-error-codes.yml`.
