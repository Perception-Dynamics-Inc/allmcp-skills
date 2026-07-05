# iiko orders — workflows

Jump to the task that matches the user's request. Steps marked **✍** write
data to iiko — confirm specifics before firing a ✍ step. The status table at
the bottom holds the values you cannot guess.

## Model note — read before acting

- **Create, read, and cancel only.** No tool updates order content or moves an
  order through states after creation — don't promise an edit.
- **Delivery address is set on create.** `iiko_create_order` takes an optional
  `delivery_point` (coordinates or a structured address) and forwards it as
  `deliveryPoint`. Pass it for courier delivery; omit it for customer pickup.
  It cannot be changed after creation — to fix a wrong address, cancel and
  recreate with the correct `delivery_point`.
- **Retries are duplicate-safe.** Every create sends a client-side order ID
  (your `idempotency_key` if given, else an auto-generated UUID). On a timeout
  the error names that ID — verify with `iiko_get_order` before retrying; a
  retry that reuses the same ID will not place a second order.
- **`iiko_list_orders` returns ALL statuses** — no status-filter parameter.
  Filter client-side. `total` is always `null` (iiko does not report a true
  count), so never read it as the number of orders; if you get back exactly
  `limit` records, split the date range and re-query.

## Task: place a new delivery order

Full prerequisite chain — do not skip steps.

1. `iiko_list_organizations()` → pick the org → `organization_id` (UUID).
2. `iiko_list_terminal_groups(organization_ids=[organization_id])` → pick the
   terminal → `terminal_group_id`.
   `organization_ids` is a **list** — `[org_id]`, not a bare string.
3. `iiko_get_menu(organization_id=organization_id)` → find products by name in
   the `products` array (use the `name` field; skip items where `isDeleted` is
   true or `isGroupModifier` is true) → collect `productId` UUIDs.
   See the menu category skill for full filtering guidance.
4. `iiko_list_stop_list(organization_ids=[organization_id])` → confirm none of
   your `productId`s appear. If a product is stopped, tell the user before
   proceeding. (`iiko_get_menu` does not include the stop list.)
5. **✍** `iiko_create_order(organization_id=..., terminal_group_id=...,
   phone="+79991234567", items=[{"productId": "<uuid>", "amount": 1}])` → `id`.
   For courier delivery add `delivery_point={"coordinates": {"latitude": ...,
   "longitude": ...}}` (or a structured address); omit it for pickup.
6. `iiko_get_order(organization_id=..., order_id=<step 5>)` → confirm
   creation and read initial `status`.

## Task: cancel an order

Always fetch the current status first — never assume cancellability from a
lean list record (lean record fields vary; only `iiko_get_order` guarantees the
`status` field).

1. `iiko_list_orders(organization_id=..., date_from="...", date_to="...")` →
   scan results to identify the target order → note its `id`.
2. `iiko_get_order(organization_id=..., order_id=<id from step 1>)` → read
   the full record and check `status`. Do not rely on any field from the lean
   list record to determine cancellability.
3. If `status` is `Delivered`, `Closed`, or `Cancelled` — stop. Tell the user
   the order is terminal and cannot be cancelled.
4. **✍** `iiko_cancel_order(organization_id=..., order_id=<step 1>,
   cancel_cause="Cancelled by customer")`.
   `cancel_cause` is free text — pass any plain string or omit it entirely.
   If iiko rejects the cancellation with a 4xx error, retry once with
   `cancel_cause` omitted; the cause text may not be accepted by this org's
   configuration.

## Task: look up orders for a date range

`iiko_list_orders(organization_id=..., date_from="2024-06-01 00:00:00",
date_to="2024-06-01 23:59:59", limit=500)` — lean records (fields vary — call
`iiko_get_order` for full detail on any specific order, including items,
delivery point, and timestamps).

## Reference — order status enum

| Status | Cancellable |
|---|---|
| `Unconfirmed` | Yes |
| `WaitCooking` | Yes |
| `ReadyForCooking` | POS-config dependent (unconfirmed in Cloud API — verify in iikoWeb) |
| `CookingStarted` – `Waiting` | POS-config dependent — attempt; surface error to user |
| `OnWay` | Unlikely |
| `Delivered` | No — terminal |
| `Closed` | No — terminal |
| `Cancelled` | No — already terminal |
