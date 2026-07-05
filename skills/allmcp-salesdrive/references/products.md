# SalesDrive products — workflows

Jump to the matching task. **✍** marks catalogue writes — confirm specifics on a
vague request before firing one.

## Model note — read before acting

- **No list, search, or browse tool exists.** `salesdrive_get_product`
  needs an exact ID (numeric or SKU); you cannot enumerate the catalogue or find
  a product by name. To act on "the product called X", read its `id` from an
  order's line items via `salesdrive_get_order` / `salesdrive_list_orders`
  (orders category). With no ID, ask for the SKU — never invent one. (To *rename*
  a known product, `salesdrive_update_product(..., updates={"name": ...})` works
  — it's only *finding* one by name that has no tool.)
- **Read field ≠ write field for price.** `get_product` returns `price`, but the
  field that *sets* price is **`costPerItem`**. SalesDrive silently ignores
  unknown keys (HTTP 200, no error), so writing `price` returns `success=True`
  with nothing changed. Round-trip: read `item.id` and `item.price` → compute →
  write `{"id": item.id, "costPerItem": "<new>"}`, keyed by the numeric `id` the
  read returned.
- **Writes never self-verify** — `success=True` is an HTTP-200 echo, and a batch
  upsert hides per-item errors. Re-read changed IDs with `salesdrive_get_product`
  before reporting success.
- **Always pass `paramsMode` / `labelMode` = `"add"` when editing part of
  `params` / `label`.** `"replace"` overwrites the whole existing set, and the
  server default when the mode is omitted is undocumented (it may wipe) — never
  rely on it. Re-read after to confirm the rest survived.
- **Create / delete** via `salesdrive_create_product` (or `salesdrive_upsert_products`
  without `id`) and `salesdrive_delete_product(s)` (≤100). No undo or restore tool
  exists, so confirm IDs before deleting. Set `category` as `{id, name}`; resolve
  the numeric `id` in the `categories` category (you must already know it).

## Task: change product prices, or sync stock

1. `salesdrive_get_product(product_id="TS-RED-44", include="summary")` → read the
   product's numeric `id` and current `price`; compute the new value per item.
2. **✍** `salesdrive_upsert_products(products=[{"id": <id from step 1>,
   "costPerItem": "330"}, ...])` — write **`costPerItem`, not `price`** (for
   stock: `stockBalance` total, or `stockBalanceByStock` `{warehouseId: qty}`).
3. `salesdrive_get_product(product_id=<id from step 1>, include="summary")` on a
   sample of the changed IDs → confirm the new value landed; the write is
   unconfirmed until then.

## Task: schedule a promotional discount (e.g. Black Friday)

`discount` is a **date-bounded object**, not a number — `discount: 20` does
nothing, and dated windows ARE supported (no UI fallback or manual toggling).

1. **✍** `salesdrive_update_product(product_id="TS-RED-44", updates={"discount":
   {"value": "20", "date_start": "2024-11-24", "date_end": "2024-11-30"}})`.
2. `salesdrive_get_product("TS-RED-44")` → confirm it stuck. `value` is a string
   of unconfirmed unit (percent vs amount) — read a discounted product, or ask
   the user, before mass-applying.

## Task: add or edit a characteristic (params)

1. `salesdrive_get_product("TS-RED-44")` → read the numeric `id` and existing
   `params`.
2. **✍** `salesdrive_upsert_products(products=[{"id": <id from step 1>, "params":
   [{"name": "Колір", "type": "select", "value": "червоний"}], "paramsMode":
   "add"}])` — `"add"` appends; the documented `type` is **`"select"`** (don't
   guess `"string"`/`"text"`; copy `type` off an existing param if unsure).
3. `salesdrive_get_product(product_id=<id from step 1>)` → confirm the new AND old
   params both survived.

## Gotchas

- `set`, `images`, and `additionalPrices` are lists of objects (like `discount`),
  never bare scalars — send the documented object shape, not a number or string.
- Currency codes, price-type names, and category ids are portal-configured and
  not enumerated by any tool — copy them off an existing record.
