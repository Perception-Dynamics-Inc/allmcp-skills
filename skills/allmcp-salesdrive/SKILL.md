---
name: allmcp-salesdrive
description: Work SalesDrive — the dominant Ukrainian e-commerce CRM — orders, products, categories, payments, reference data — through the AllMCP hub. Use when the user asks to read or change anything in their SalesDrive account (orders, catalog, prices, stock, payments, statuses) via AllMCP. NOT for calling SalesDrive's HTTP API directly with your own code, or for admin-panel settings (users, integrations, status/method configuration).
---

# SalesDrive via AllMCP

SalesDrive is the dominant e-commerce CRM in Ukraine. Through AllMCP your
agent gets 18 namespaced tools (`salesdrive_list_orders`,
`salesdrive_upsert_products`, …) organized into five categories. This file is
orientation and routing; the real playbooks are in `references/` — read the
matching one **before** chaining tools in a category you haven't used this
session.

Platform mechanics (endpoint, connecting, hidden tools, quota) live in the
`allmcp` skill — install it alongside this one.

## Connect

- Provider key: `salesdrive` (exactly this, snake_case).
- Multi-field credential — pass all three as `extra_fields` of
  `connect_provider`:
  `extra_fields={"domain": "...", "form_key": "...", "account_key": "..."}`.
  The connector rejects the key pair without the domain.
- `domain` is the **bare subdomain** — `shop-name` from
  `shop-name.salesdrive.me`, never the full URL.
- `form_key` — the Form API key; covers orders, products, categories, and
  reference data. The user generates it in their SalesDrive admin panel at
  `https://{domain}.salesdrive.me/ru/index.html?formId=1#/web-form/new-api-key-form`.
- `account_key` — the Account API key; covers payments. Generated at
  `https://{domain}.salesdrive.me/ru/index.html?formId=1#/web-form/new-api-key-account`.
- Two keys, two scopes: if payments error while orders work fine, suspect a
  bad `account_key`, not a broken account — reconnect with a fresh one.
- Both keys are live credentials — never echo them back in full or log them.

## The three rules that prevent most failures

1. **There is almost no lookup surface — plan ID sourcing before acting.**
   Only statuses, payment methods, and delivery methods resolve name→ID (the
   `reference` category). Products have no list/search, categories have no
   read at all, and orders have no free-text search — every other ID comes
   from an existing order row, the SalesDrive UI, or the user.
2. **Writes never self-verify.** `success=True` is an HTTP-200 echo, and
   unknown fields are silently ignored — a wrong field name "succeeds" while
   changing nothing. Re-read the record after any write that matters before
   reporting success.
3. **No idempotency keys.** Always pass `external_id` when creating orders
   and `unique_id` when creating payments; if a create errors ambiguously,
   list-and-check for the record first — a blind retry duplicates the order
   or double-records the money.

## Categories → playbooks

Read the reference file before working in a category; each one carries the
unguessable field splits (read `price` vs write `costPerItem`), the default
enums, and the ID flow between categories.

| Category | Read | Covers |
|---|---|---|
| `orders` | [references/orders.md](references/orders.md) | Find/create/update orders, status moves, revenue rollups, tracking numbers |
| `products` | [references/products.md](references/products.md) | Get by ID, create/update/delete, batch upsert — **no list or search** |
| `categories` | [references/categories.md](references/categories.md) | Create, nest, rename, delete product categories — **write-only** |
| `reference` | [references/reference.md](references/reference.md) | Statuses, payment + delivery methods — the name→ID resolver (read-only) |
| `payments` | [references/payments.md](references/payments.md) | Record payments, attach to orders, period totals |

## Boundaries worth knowing up front

- **Products are get-by-ID only.** No tool lists or searches the catalog —
  read a product's `id` off an order's line items, or ask the user for the
  SKU. Batch tools (`salesdrive_upsert_products`,
  `salesdrive_delete_products`) cap at 100 items per call.
- **Categories are write-only.** No list/get/search exists, and upsert never
  returns the new category's id — valid ids come only from the SalesDrive UI
  or YML export. Never create a category planning to "fix it later".
- **Payments are record + read only.** No edit or delete — a wrong payment
  can only be offset by a new one, so confirm amount and order first.
- **Orders have no delete and no free-text search.** "Delete" means moving
  the order to the Deleted status; find a customer's order via a date-window
  list plus client-side matching.
- **Reference data is read-only and tenant-editable** — resolve status IDs
  and method codes at runtime; never hardcode the defaults.
