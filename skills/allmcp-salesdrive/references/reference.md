# SalesDrive reference — workflows

This category is a **name↔ID resolver**, not an action surface. The three lists —
`salesdrive_list_statuses`, `salesdrive_list_payment_methods`, `salesdrive_list_delivery_methods` —
each return the whole set in one no-argument call (no get-by-ID, no search); match a row by its `name`
client-side, then feed its `id`/`parameter` into the `orders` category. Resolve here, act there; a
**✍** step mutates orders — confirm specifics first if the request is vague.

## Model note — read before acting

- **Read-only.** You cannot create, rename, reorder, or delete a status, payment, or delivery
  method — those are admin-UI settings, not API calls. Don't promise a rename.
- **Status `type` is an integer 1–4, not a string.** Its meaning is undocumented; infer a status's
  lifecycle from the row's `name`, never from a guessed string.
- **IDs this category does NOT resolve** (no list tool — read off an existing order row or ask the user):
  manager `userId`, rejection-reason, organization, order-type, warehouse, currency.

## Task: list or move an order set by status

1. `salesdrive_list_statuses()` → match the status `name` (e.g. Shipped, Sale) → `id`. Resolve at
   runtime — tenants rename statuses, so don't trust the defaults below.
2. List: `salesdrive_list_orders(filter_status_ids=[<id>])`. Move: **✍**
   `salesdrive_update_order(order_id="10472", status_id=<id>)`. Both in `orders`.

## Task: create or filter an order by delivery / payment method

1. `salesdrive_list_delivery_methods()` → find the carrier by `name` (e.g. Nova Poshta) → read **that
   row's** `parameter`; same with `salesdrive_list_payment_methods()` (COD vs pay-in-hand). Never hardcode
   `novaposhta`/`postpay` — they're example codes a tenant may customize, so a literal `parameter == "postpay"` match can silently miss.
2. Create: **✍** `salesdrive_create_order(..., shipping_method=<delivery param>, payment_method=<payment
   param>)`. Filter: `salesdrive_list_orders(filter_payment_method=...)` / `filter_shipping_method=...`. Both in `orders`.

## Reference — IDs & enums (defaults; resolve at runtime, admins edit these)

**Statuses** (`id` · Ukrainian name → meaning): 1 Новий → New · 2 Підтверджено → Confirmed · 3 На
відправку → Ready to ship · 4 Відправлено → Shipped · 5 Продаж → Sale · 6 Відмова → Rejected · 7
Повернення → Return · 8 Видалений → Deleted.

**Status-filter sentinels** (`filter_status_ids`, not real statuses): `__NOTDELETED__` (server
default — all but Deleted), `__ALL__` (include Deleted).

**`parameter` codes are tenant-editable — read them live, never hardcode.** Two payment codes whose
meaning isn't obvious: `postpay` = COD (cash on delivery), `tocard` = pay-by-card.
