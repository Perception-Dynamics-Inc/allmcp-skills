# SalesDrive payments — workflows

Jump to the task matching the request. Steps marked **✍** record money movement —
for a vague request, confirm amount, order, and direction before firing one.

## Model note — read before acting

- **Record + read only.** No edit or delete tool exists; a wrong sum/date/description
  cannot be corrected, only offset by a new payment. Never promise to edit or remove one.
- **`organization_id` and `account_number` are required to create, and no tool lists
  them.** Read them off an existing payment in `include='full'`, where they sit in the
  nested `organization` / `organizationAccount` blocks — NOT as flat
  `items[].organization_id`. If no prior payment exists, ask the user.
- **A payment has no top-level `orderId`, `status`, or `method`** — only `sum`, `date`,
  `type`, and `paymentBreakdown[]`. The order it is attached to lives in
  `paymentBreakdown[].order.id`, visible only in `include='full'`; the `summary`
  keep-list advertises `orderId` but it is usually absent, so don't scan for it.
- **No order filter, no users directory.** "All payments for order X" is not a
  server-side filter — list by date and inspect each `paymentBreakdown`. A payment's
  manager `userId` cannot be resolved to a name. Resolve an `order_id` /
  `order_external_id` in the `orders` category (`salesdrive_list_orders`,
  `salesdrive_get_order`).

## Task: record a payment and attach it to an order

1. Source `organization_id` + `account_number` per the Model note.
2. **✍** `salesdrive_create_payment(organization_id="1",
   payment_datetime="2026-06-30 14:00:00", account_number="UA12...", payment_sum="1500",
   order_id="14820", unique_id="ord14820-20260630")`. Always pass a `unique_id` —
   SalesDrive has no idempotency keys, so it is your only dedupe handle. With no order
   ID, bind by payer via `auto_attach_to_order_type="lastname_and_sum"` +
   `payer_last_name=...`.
3. If the tool errors with "did not return a payment ID", the payment **likely recorded
   anyway** — do NOT re-create blindly (that double-records). Verify with
   `salesdrive_list_payments(include="full", ...)` — `unique_id` is dropped from the
   default `summary` shape, so a summary scan can't match it; in `full` match by date +
   `unique_id`/sum. Re-create only if it is genuinely absent.

## Pagination & period totals

For a period sum, don't page and add up rows — read `extra` (`count`, `sum`) off
`salesdrive_list_payments(filter_date_from="2026-06-01 00:00:00",
filter_date_to="2026-06-30 23:59:59", payment_type="incoming")`. To walk the full log,
advance `page` (1-based) until a page returns fewer than `limit` rows; `next_cursor` is
unreliable — it can be `null` while more pages remain, so don't treat null as the end.
