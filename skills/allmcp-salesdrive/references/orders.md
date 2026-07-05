# SalesDrive orders — workflows

Jump to the task matching the user's request. Steps marked **✍** create or change an
order — if the request is vague, confirm specifics first. Resolve unknown status,
shipping, and payment codes via the `reference` category (see the end of this doc).

## Model note — read before acting

- **No delete.** "Deleting" an order is **✍** `salesdrive_update_order(order_id="...", status_id="8")`
  (soft-delete); only `filter_status_ids=["__ALL__"]` then lists it.
- **No free-text search.** No filter matches a name, phone, email, SKU, or product — find a
  person via the date-window + client-side match in the *find an order* task below.
- **No lookup for opaque IDs.** `filter_user_id` (manager), `rejection_reason_id`,
  `organization_id`, `stock_id`, `filter_type_id`, `filter_form_id`, `filter_campaign_id`
  have no resolution tool — the caller must already know them; `reference` does NOT help.
- **Products/categories aren't reachable here.** Never promise to list, search, or rename
  products or categories. Line items carry the product `id` → `salesdrive_get_product`
  (`products` category) for catalogue detail.
- **`payment_date` marks the sale date — it does NOT record money.** Log a payment with
  `salesdrive_create_payment` (`payments` category).

## Task: report realized revenue for a period

The default view (`__NOTDELETED__`) includes New, Rejected, and Return, and `paymentAmount`
is the invoiced order TOTAL — summing it over the default view over-counts revenue.

1. `salesdrive_list_orders(filter_status_ids=["5"], filter_payment_date_from="2026-06-01",
   filter_payment_date_to="2026-06-30", limit=100)` — completed Sales only, over the sale date.
2. Report `extra.paymentAmount` as gross sales — one request, no paging. `payedAmount` is cash
   actually collected and runs lower on partly-paid orders; read it per order (page via
   `next_cursor` until null) when the user wants cash received.
3. State the basis you reported: completed Sales, invoiced total vs cash collected.

## Task: find an order by customer or source

1. By customer name: list a date window (`filter_order_time_from`/`_to`, `limit=100`) and match on
   `contact` in `items`. Empty results aren't proof of absence — the window may be too narrow.
2. By UTM campaign (Instagram / Google Ads promo): list the window with `include="full"`, read the
   `utmSource`/`utmMedium`/`utmCampaign` values off the rows, then re-run filtering on
   `filter_utm_source`/`filter_utm_medium`/`filter_utm_campaign` (they AND-combine).
3. By source URL: `sajt`/`utmPage` are stripped from every list response (even `include="full"`),
   so you can't discover a tenant's source URLs from rows — `salesdrive_get_order(order_id=X,
   efficiency_on=False)` is the only view that shows `sajt`, but it needs the ID already. Pass
   `filter_sajt` only with a URL the user supplies; a wrong URL silently matches zero orders.

## Task: create an order from a lead

1. **✍** `salesdrive_create_order(first_name="Olena", phone="+380501234567",
   products=[{"id": 4471, "amount": 2, "costPerItem": "350"}], shipping_method="novaposhta",
   novaposhta={...}, external_id="WEB-5512")` → new order `id`.
2. Fill only the block matching `shipping_method`; Rozetka's block is `rozetka_delivery`, not `rozetka`.
3. SalesDrive has no idempotency keys: always pass `external_id`. If the call errors or returns
   no `id`, do NOT blind-retry — run `salesdrive_list_orders(filter_external_id="WEB-5512")` first;
   a second create spawns a duplicate order.

## Task: attach a tracking number after shipping

TTN is the only delivery field patchable after creation, and it is silently dropped if the order
has no matching delivery block.

1. **✍** `salesdrive_update_order(order_id="37193", status_id="4", novaposhta_ttn="20450012345678")`.
2. `salesdrive_get_order(order_id=37193)` → confirm the TTN landed on the Nova Poshta block before
   telling the user it shipped. `success=True` alone does not prove the TTN stuck.

## Reference — resolving codes

The status map and method codes in the tool schemas are SalesDrive **defaults**; portals add
custom statuses (IDs >8) and rename labels. Resolve unknown values via the `reference` category:
`salesdrive_list_statuses`, `salesdrive_list_payment_methods`, `salesdrive_list_delivery_methods`.
