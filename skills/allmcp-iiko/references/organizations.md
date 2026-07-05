# iiko organizations — workflows

This category is the entry point for the entire iiko connector — every `menu`
and `orders` tool requires an `organizationId`.

## Model note — read before acting

- **`organizationId` is the universal prerequisite.** Skip `iiko_list_organizations`
  and the next iiko call fails with a provider-level error.
- **Terminal groups have no `type` or `kind` field.** Items expose `id`,
  `name`, and injected `organizationId`; `isDeleted` and `externalRevision`
  appear if present. Identify the delivery terminal by its `name` (set in
  iikoWeb, e.g. "Delivery Terminal"). If names are ambiguous, present all
  non-deleted groups and ask the user — never filter by an invented type enum.
- **Always filter `isDeleted: true` groups before use.** Passing a deleted
  group's `id` to `iiko_create_order` fails at the iiko API level.
- **No sort order guaranteed.** Do not infer which org or group is newest from
  list position — present names and confirm with the user.
- **Read-only.** No create, update, or delete tools exist; orgs and terminal
  groups are iikoWeb configuration.

## Task: get an organizationId before any other iiko call

1. `iiko_list_organizations()` → each `items[].id` is an `organizationId`.
2. For multi-location users, collect all needed IDs in one call — menu and
   order tools accept a list.

## Task: find the terminalGroupId for order creation

`iiko_create_order` requires both `organization_id` and `terminal_group_id`.

1. `iiko_list_organizations()` → `org_id`.
2. `iiko_list_terminal_groups(organization_ids=[org_id])` → flat group list.
3. Drop any item where `isDeleted` is `true`.
4. Match the delivery terminal by `name`. If ambiguous, present options and ask.
5. Use the chosen item's `id` (the `terminalGroupId`) — not `organizationId`.

## Gotchas

- Extended org fields (`currencyIsoName`, `deliveryServiceType`) may be absent;
  the tool omits `returnAdditionalInfo`. Acknowledge the gap rather than
  asserting a value.
