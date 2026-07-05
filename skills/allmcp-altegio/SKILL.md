---
name: allmcp-altegio
description: Work Altegio (alteg.io, ex-YClients) — salon appointments, clients, staff, services catalog, manual finance ledger — through the AllMCP hub. Use when the user asks to read or change anything in their Altegio salon (book/reschedule/cancel appointments, look up or edit clients, manage staff, quote services, record ledger transactions) via AllMCP. NOT for calling Altegio's REST API directly with your own code, or for checkout/refunds/fiscal-register (KKM) operations, which this connector does not perform.
---

# Altegio via AllMCP

Altegio is an online-booking, CRM, and POS platform for salons, barbershops,
and service businesses. Through AllMCP your agent gets namespaced tools
(`altegio_create_appointment`, `altegio_list_clients`, …) organized into
categories. This file is orientation and routing; the real playbooks are in
`references/` — read the matching one **before** chaining tools in a category
you haven't used this session.

Platform mechanics (endpoint, connecting, hidden tools, quota) live in the
`allmcp` skill — install it alongside this one.

## Connect

- Provider key: `altegio` (exactly this, snake_case).
- Credentials: **three fields**, passed together as
  `extra_fields={"partner_token": ..., "user_token": ..., "company_id": ...}`
  on `connect_provider`. All three are required.
- Where the user gets them — Altegio admin, **Integrations → Developer's
  account**:
  - `partner_token` — the integration API key, under **Account settings**.
  - `user_token` — via **Add app → Access to API**; the user must tick every
    access right they want the agent to have (appointments/records, clients,
    staff, finances) when generating it.
  - `company_id` — the numeric salon/location ID, visible in the Altegio
    admin URL (or via `altegio_list_companies` once connected). Must be a
    positive integer.
- Both tokens are live credentials — never echo them back in full or log
  them; if pasted somewhere public, tell the user to regenerate them.
- The connected `company_id` is the default salon for every call; tools also
  accept a per-call `company_id` override for multi-location businesses.

## The three rules that prevent most failures

1. **`company_id` scopes everything.** Every tool defaults to the salon you
   connected with. If the account has multiple locations, resolve the right
   one with `altegio_list_companies` first and pass `company_id` explicitly —
   otherwise you silently read/write the wrong salon.
2. **Resolve IDs from their own categories before booking.** A booking needs
   a `staff_id` (from `altegio_list_staff`) and `service_ids` (from
   `altegio_list_services`) — never guess them or scrape them from the
   appointments list. And there is **no availability check**:
   `altegio_create_appointment` posts straight to the schedule, so list the
   staff member's existing appointments for that day first or you
   double-book.
3. **A 403 means a missing access right, not a bad ID.** The `user_token`
   only covers the rights ticked when it was generated; "No rights to manage
   location" is permanent until the user re-mints the token with that right
   added. Don't retry with different IDs.

## Categories → playbooks

Read the reference file before working in a category; each one carries the
unguessable enums (attendance statuses, ledger article types), the ID-flow
between tools, and the boundaries.

| Category | Read | Covers |
|---|---|---|
| `appointments` | [references/appointments.md](references/appointments.md) | Book, reschedule, no-show/cancel, delete records; attendance enum |
| `services` | [references/services.md](references/services.md) | Read-only catalog — service IDs, durations, price ranges for booking |
| `clients` | [references/clients.md](references/clients.md) | Search by name/phone, create, update, dedupe, delete |
| `staff` | [references/staff.md](references/staff.md) | Roster, hire, dismiss/hide — the `staff_id` resolver for everything else |
| `finances` | [references/finances.md](references/finances.md) | Manual ledger — transactions CRUD, payment accounts, expense articles |

The `companies` category is thin — one tool, `altegio_list_companies`, which
lists locations and their IDs; discover it with
`describe_category("altegio", "companies")`.

## Boundaries worth knowing up front

- **Finances is a manual bookkeeping ledger only** — it does not run
  checkout, mark visits paid, issue refunds, or touch the fiscal register
  (KKM) or inventory. Route those requests to the salon's register flow.
- The services catalog is **read-only** — no creating, renaming, or
  repricing services, and no free-slot/availability lookup anywhere.
- No staff schedule surface (shifts, days off, rota) and no staff↔service
  linking; dismiss staff with `fired`/`hidden` flags rather than deleting.
- Deletes (clients, staff, appointments, transactions) are hard and
  irreversible — prefer the status-based alternatives the references name.
- Durations (`seance_length`) are in **seconds** across appointments and
  services — 3600 is one hour, not an hour count.
