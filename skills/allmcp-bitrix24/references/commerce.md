# Bitrix24 commerce — workflows

Six read-only tools: CRM product catalog, online store orders, payments. For
any change (prices, order status, refunds), surface the data and direct the
user to Bitrix24 itself. Jump to the task matching the user's request.

## Model note — read before acting

- **No stock or inventory data exists here.** Product payloads carry no
  quantity field — inventory lives in Bitrix24's separate inventory API, which
  is not wrapped. Answer stock questions with a definitive "not retrievable
  through these tools", never "N units if the field exists".
- **Two field-casing families in one category.** Products use SCREAMING_SNAKE
  keys: `ID`, `NAME`, `PRICE`, `CURRENCY_ID` (not `currency`), `ACTIVE`.
  Orders and payments use camelCase: `id`, `dateInsert`, `statusId`, `payed`,
  `paySystemName`. Key all client-side filtering to the right family.
- **`include="summary"` and `include="full"` return identical payloads on all
  six tools** (passthrough summarizer). Never re-fetch with the other value
  expecting more or fewer fields; only `efficiency_on=False` on the `get_*`
  tools changes the payload (raw debug form).
- **`bitrix24_list_orders` returns unfiltered, unordered pages.** Any "orders
  since / orders where…" job means reading every page, then filtering
  client-side — see that recipe before paginating.
- Flags are `"Y"`/`"N"` strings: order `payed` (paid in full) / `canceled` /
  `deducted` (shipped), payment `paid`, product `ACTIVE`.

## Task: look up a product's price or details

1. `bitrix24_list_products(query="ErgoMax")` → match on `NAME`, take `ID`.
2. `bitrix24_get_product(product_id=<step 1>)` → `PRICE`, `CURRENCY_ID`,
   `ACTIVE`, `SECTION_ID`. `MEASURE` is a numeric unit-of-measure ID, not a
   string; custom `PROPERTY_*` fields never appear in list output.

No product filters beyond name exist — for audits by price, section, or
active flag, page the whole catalog and filter client-side.

## Task: check whether an order is paid, canceled, or shipped

1. `bitrix24_get_order(order_id=2041)` → answer from the three flags:
   `payed`, `canceled`, `deducted`.
2. `statusId` is a portal-configured letter code with no lookup tool here —
   report it verbatim or ask the user what their codes mean; never assert a
   meaning (only `"N"` = initial status is a system default).

## Task: see what was in an order and who bought it

1. `bitrix24_get_order(order_id=2041)` → line items under `basketItems`,
   buyer data under `clients` and `propertyValues` (not `buyer`/`properties`),
   shipping under `shipments`, attached payments under `payments`, order
   total under `price`, human-facing order number under `accountNumber`
   (distinct from the numeric `id`).
2. `userId` is a numeric ID and may be a store buyer account rather than a
   portal employee — try `bitrix24_list_users` (organization category); if
   there is no match, report the raw ID and say it can't be resolved here.

## Task: reconcile or investigate payments on an order

1. `bitrix24_list_payments(order_id=2041)` → per payment: `sum`, `paid`,
   `paySystemName`, `datePaid`.
2. For a failed or odd payment: `bitrix24_get_payment(payment_id=<step 1>)`
   → `psStatus` / `psStatusCode` / `psStatusDescription` are raw codes echoed
   from the external payment provider — interpret them per that provider, not
   as Bitrix24 enums.

## Task: find orders matching a date, payment, or status criterion

1. Page `bitrix24_list_orders(start=0)` through ALL pages (`next_cursor` →
   `start` until null). Never stop early because a page "looks old" — result
   order is not guaranteed newest-first, and on an ascending portal an
   early stop silently drops every qualifying order.
2. Filter and sort client-side on `dateInsert`, `payed`, `statusId`, etc.
   (camelCase keys).
3. Only then call `bitrix24_get_order` on the survivors — the full order
   graph is large; don't loop it over the whole order book.

## Task: work the sales side of a store order

Store orders auto-create CRM deals. For the deal (stage, owner, follow-up),
use `bitrix24_list_deals` / `bitrix24_get_deal` in the `crm` category.
Product rows attached to deals or estimates are not reachable from commerce —
this catalog is the dictionary they reference by ID.

## Pagination

Every list tool returns a fixed 50 per page, not adjustable. Page
sequentially — the portal rate-limits around 2 requests/second shared across
the webhook. `total` can drift mid-scan; treat it as an estimate.

## Gotchas

- Orders and payments require the `sale` webhook scope AND an administrator
  user; products need only `crm`. If products work but order/payment calls
  fail with access errors, have the user re-issue the webhook from an admin
  account with the `sale` permission.
- A missing order or payment ID surfaces as an HTTP-400 error reading "order
  does not exist" / "payment does not exist" — treat it as a normal
  not-found, not an outage.
- The catalog rides Bitrix24's legacy CRM product API: variations/SKUs and
  per-price-type prices are invisible, and `SECTION_ID` is a bare numeric ID
  with no section-lookup tool.
