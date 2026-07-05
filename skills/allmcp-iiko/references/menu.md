# iiko menu — workflows

Two read-only tools: `iiko_get_menu` (full nomenclature) and `iiko_list_stop_list`
(unavailable items). Both need an organization UUID from `iiko_list_organizations` —
`iiko_get_menu` takes `organization_id` (string); `iiko_list_stop_list` takes
`organization_ids` (list).

## Model note — read before acting

- **Filter `products` before presenting.** Always apply `isDeleted == false AND
  isIncludedInMenu == true AND isGroupModifier == false` — soft-deleted items and modifier
  containers always appear in the raw array alongside orderable dishes.
- **Stop lists are terminal-group-scoped, not restaurant-scoped.** A dish stopped at the
  bar may still be available at the kitchen. Never report a dish as globally unavailable
  from a single stop-list row — it is globally stopped only when ALL terminal group entries
  for that `productId` show `balance: null` or `balance: 0`.
- **`balance: null` and `balance: 0` both mean unavailable.** Never interpret `null` as
  "unknown / possibly available."
- **Cross-reference:** `stop_item["productId"] == product["id"]` (both UUIDs). Never
  match on `code` (iiko's human-readable internal string).
- **`terminalGroupId` values are opaque UUIDs.** To show human-readable station names,
  call `iiko_list_terminal_groups` (organizations category) and join on `id`.
- **Order creation belongs to `orders` category.** Product `id` from `iiko_get_menu`
  flows into `iiko_create_order` as items `[{"productId": "<uuid>", "amount": N}]`
  (camelCase — `product_id` is wrong and will fail).
- **`revision`** in the `iiko_get_menu` response is the current menu version integer —
  compare across calls to detect changes before re-fetching the full payload.

## Task: browse the current menu

1. `iiko_get_menu(organization_id=<uuid>)`.
2. Filter `item["products"]` to `isDeleted == false AND isIncludedInMenu == true AND
   isGroupModifier == false`. Join `parentGroup` UUID to `item["groups"][*]["id"]` for
   category hierarchy; prices are in each product's `sizePrices` list.
3. Summarize by group — the raw response can be large; do not dump it unfiltered.

## Task: check unavailable items / pre-flight before ordering

1. `iiko_list_stop_list(organization_ids=[<uuid>, ...])`.
2. Group results by `productId`. A dish is globally unavailable only when **all** terminal
   group entries for it show `balance: null` or `balance: 0`; if stopped in some groups
   but not others, show the user which groups have it stopped.
3. Cross-reference to menu: `stop_item["productId"] == product["id"]`.
4. To show station names instead of UUIDs: call `iiko_list_terminal_groups` and join
   on `terminalGroupId`.

## Task: find a dish by name and get its productId

1. `iiko_get_menu(organization_id=<uuid>)` → filter products (see above) → search by
   `name` client-side → extract `id` (UUID).
2. Optionally check stop list (task above) before passing to `iiko_create_order`.
