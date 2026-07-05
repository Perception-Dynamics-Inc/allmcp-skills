---
name: allmcp-iiko
description: Work iiko — restaurant organizations, menus, stop lists, and delivery orders — through the AllMCP hub. Use when the user asks to browse an iiko menu, check item availability, or create, track, or cancel delivery orders via AllMCP. NOT for calling the iiko Cloud API directly with your own code, or for iikoWeb back-office administration (editing menus, managing terminals, staff, or finances).
---

# iiko via AllMCP

iiko is a restaurant automation and delivery-management platform. Through
AllMCP your agent gets namespaced tools (`iiko_get_menu`,
`iiko_create_order`, …) organized into three categories. This file is
orientation and routing; the real playbooks are in `references/` — read the
matching one **before** chaining tools in a category you haven't used this
session.

Platform mechanics (endpoint, connecting, hidden tools, quota) live in the
`allmcp` skill — install it alongside this one.

## Connect

- Provider key: `iiko` (exactly this).
- Credential: the iiko Cloud API **apiLogin**, passed as the single
  `api_key` field of `connect_provider`. The user copies it from
  **iikoWeb → Settings → API** (their personal API login). It is not a
  password and never expires unless revoked in iikoWeb.
- That's the only credential. AllMCP exchanges the apiLogin for a
  short-lived bearer token behind the scenes and refreshes it
  automatically — never ask the user for tokens or passwords.
- The apiLogin is still a live credential — don't echo it back in full or
  log it; if it leaks, the user revokes it in iikoWeb.

## The two rules that prevent most failures

1. **Everything hangs off `organizationId`.** Call
   `iiko_list_organizations` first — every menu and orders tool needs one.
   Order creation chains further: `iiko_list_terminal_groups` for the
   `terminal_group_id`, `iiko_get_menu` for `productId` UUIDs, then
   `iiko_create_order`. All IDs are opaque UUIDs — never guess or invent
   one, and never pass a name where an ID is expected.
2. **Raw lists include items you must not use.** Deleted terminal groups,
   soft-deleted products, and modifier containers all appear in the raw
   payloads, and a dish on the stop list will fail at order time. Filter
   per the category playbook and check `iiko_list_stop_list` before
   promising an item is orderable.

## Categories → playbooks

Read the reference file before working in a category; each one carries the
ID flow between tools, the filtering rules, and the boundaries (including
the order status enum you cannot guess).

| Category | Read | Covers |
|---|---|---|
| `organizations` | [references/organizations.md](references/organizations.md) | Organizations and terminal groups — the ID source for everything else |
| `menu` | [references/menu.md](references/menu.md) | Menu nomenclature (products, groups, prices) and stop lists |
| `orders` | [references/orders.md](references/orders.md) | Delivery orders — list, detail, create, cancel |

## Boundaries worth knowing up front

- `organizations` and `menu` are **read-only**; the only writes anywhere
  are `iiko_create_order` and `iiko_cancel_order`.
- There is **no order update** tool — an order's items, address, and phone
  are fixed at creation. To fix a mistake, cancel and recreate.
- Cancellation only works while the order is still in a cancellable state;
  `Delivered`, `Closed`, and `Cancelled` orders are terminal.
- Scope is **delivery orders only** — no table/dine-in orders,
  reservations, payments, or back-office edits (menus and terminals are
  configured in iikoWeb, not through these tools).
