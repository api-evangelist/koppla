---
name: Manage koppla tickets and obstructions
description: Create, filter, update and close koppla tickets (site issues and
  obstructions), attach evidence, and export a ticket list.
api: graphql/koppla-graphql.graphql
endpoint: https://api.koppla.de/api/graphql/v1
operations:
  - Query.ticketsV2
  - Query.ticket
  - Query.ticketChanges
  - Mutation.createTicket
  - Mutation.updateTicket
  - Mutation.createTicketStatusUpdate
  - Mutation.createTicketAttachment
  - Mutation.createTicketExport
generated: '2026-07-19'
method: generated
source: graphql/koppla-graphql-introspection.json
---

# Manage koppla tickets and obstructions

Tickets in koppla capture site issues, obstructions and change events attached
to a construction project. This skill covers the full ticket lifecycle.

> Enterprise-contract API, no public documentation. Bearer token required;
> unauthenticated calls return `extensions.code = UNAUTHENTICATED`.

## Use ticketsV2, not tickets

`Query.tickets` is **deprecated** (reason: "Contributor groups"). Always use
`Query.ticketsV2`, which filters by `project` and `contributorGroups` rather
than the retired subcontractor model.

```graphql
query($project: ID!, $after: String) {
  ticketsV2(project: $project, first: 50, after: $after,
            statusIn: ["OPEN"], orderBy: ["-createdAt"]) {
    pageInfo { hasNextPage endCursor }
    edges { node { id title status createdAt } } } }
```

Page with `first` + `after` until `pageInfo.hasNextPage` is false.

## Deprecated ticket fields — avoid

Seven `TicketNode` fields are deprecated. Most are marked `TITO`:
`extendedDeadline`, `type`, `internalId`, `cost`, `tenantTradeVariation`,
`upcomingDeadline`. Also `TicketNode.subcontractor` → use `contributorGroup`.
Selecting these still works today but is on koppla's removal path.

## Create and update

Every mutation takes one `input` argument; resolve its exact shape from
`graphql/koppla-graphql.graphql`.

- `Mutation.createTicket(input:)` — open a ticket.
- `Mutation.updateTicket(input:)` — amend one. `updateTickets` handles a batch.
- `Mutation.createTicketStatusUpdate(input:)` — append a status change
  (this is how a ticket progresses and closes); `createTicketStatusUpdates`
  batches them.
- `Mutation.createTicketAttachment(input:)` — attach evidence via the `Upload`
  scalar (multipart request, not plain JSON).

**No idempotency.** Nothing in the schema deduplicates a retried
`createTicket` — a repeat call opens a second ticket. On timeout, re-query
`ticketsV2` with a tight filter to check whether the ticket exists before
retrying. `clientMutationId` correlates a response; it does not protect a write.

## Audit trail

`Query.ticketChanges` returns the change history. The `TicketChangeUnion`
resolves to `TicketStatusUpdateNode`, `TicketValueChangeNode`,
`TicketAttachmentChangeNode` or `TicketDeletedNode` — use an inline fragment
per member:

```graphql
... on TicketStatusUpdateNode { id status createdAt }
```

Use `Query.modifiedTickets` to sync only what changed since your last poll
rather than re-reading the whole list.

## Export

`Mutation.createTicketExport(input:)` produces an export; templates come from
`Query.ticketExportTemplates`. Exports may be asynchronous — poll rather than
assuming the artifact is ready on return.

## Errors

HTTP 200 with `errors[]`; check `extensions.code`. Gateway rejections arrive as
`{"message":"Missing Authentication Token"}` with HTTP 403. See
`errors/koppla-error-codes.yml`.
