---
name: allmcp-kommo
description: Work Kommo — leads, contacts, companies, pipelines, tasks, and the unsorted inbox — through the AllMCP hub's kommo_* tools. Use when: the user's CRM lives on *.kommo.com and they want to read or change sales data via AllMCP. NOT for: amoCRM tenants (amocrm.ru) — separate platform, separate skill (allmcp-amocrm).
---

# Kommo via AllMCP

Kommo (`*.kommo.com`) is the international sales-pipeline CRM — the 2022
global rebrand of amoCRM's international edition. Kommo and amoCRM share one
v4 API family but run on **separate platforms**: separate hosts, separate
logins, separate data. In AllMCP, Kommo tools are namespaced `kommo_*`
(`kommo_list_leads`, `kommo_create_lead`, …) and mirror the amoCRM surface
one-for-one, routed to `*.kommo.com`.

Platform mechanics (endpoint, connecting, hidden tools, quota) live in the
`allmcp` skill — install it alongside this one.

## Connect

- Provider key: `kommo` (exactly this, verbatim).
- Auth is **OAuth2**: call `connect_provider(provider_key="kommo")` with no
  credentials. The response is `action_required` with a consent URL — hand
  that URL to the user and stop. They open it, sign in at kommo.com, and
  approve; the connection completes on its own. Never ask the user to paste
  API keys or secrets for this flow — AllMCP owns the consent and keeps the
  token refreshed.
- Alternative for clients that can't do a browser round-trip: a long-lived
  token generated in Kommo under **Settings → Integrations → Create
  integration → Long-lived token**, passed as
  `extra_fields={"subdomain": "your-company", "access_token": "..."}` —
  subdomain is the bare label, without `.kommo.com`.

## Model note

- **Everything is a numeric ID** — leads, contacts, pipelines, statuses,
  users, custom fields. Never pass names where IDs are expected; resolve
  users via `kommo_list_users` and records via the matching `list_*` tool.
- **Pipeline and status IDs are portal-specific configuration.** There is no
  universal "Qualified" stage — call `kommo_list_pipelines` first and use the
  IDs it returns before creating leads or moving stages.
- **Entities are linked, not embedded.** A deal is a *lead* linked to
  *contacts* and a *company*; creating a lead does not create its people.
  Use `kommo_link_lead` (or `kommo_create_lead_complex` for one-shot
  lead+contact+company creation) to wire them together.
- **Inbound enquiries land in the unsorted inbox**, not in the pipeline.
  Form/chat/call submissions sit as unsorted items until accepted
  (`kommo_accept_unsorted` — creates the real lead + contact) or declined.
- **Custom fields carry most business data.** Read
  `kommo_list_custom_fields` to learn field IDs and enum values before
  writing them on leads or contacts.

## Categories

Twelve categories; only `leads`, `contacts`, and `companies` are advertised
right after connecting. Before first use of any category this session, call
`describe_category("kommo", "<category>")` — it enables the tools and returns
the workflow guide.

| Category | Covers |
|---|---|
| `leads` | Sales-pipeline opportunities — list, create, update, link, tag |
| `contacts` | Individual people |
| `companies` | Organisations linked to leads/contacts |
| `customers` | Loyalty / repeat-purchase customer records |
| `tasks` | Assigned to-dos with due dates |
| `notes` | Free-form notes on leads, contacts, customers |
| `pipelines` | Pipelines and stages — the sales-process configuration |
| `catalogs` | Custom record types (products, SKUs) and their elements |
| `custom_fields` | Custom field definitions and metadata |
| `users` | Workspace members and roles — the name→ID resolver |
| `unsorted` | Incoming-inbox leads from forms, chats, calls |
| `comms` | Calls, talks, events, webhook subscriptions |

## Task: Turn an inbound form enquiry into a staged deal

1. `kommo_unsorted_summary` → the inbox holds 3 new items today;
   `kommo_list_unsorted` shows one form submission from "Acme Signage".
2. `kommo_get_unsorted` on that item — read the embedded enquiry details
   (name, email, message) before deciding.
3. ✍ `kommo_accept_unsorted` — accepting creates the real lead and contact
   and returns their new numeric IDs.
4. `kommo_list_pipelines` → find the "Sales" pipeline ID and its
   "Qualified" status ID (IDs differ per portal — never hardcode).
5. ✍ `kommo_update_lead` — move the new lead to the Qualified status, set
   `price` to the quoted 4800.
6. `kommo_list_users` to resolve the owner "Dana" to her user ID, then
   ✍ `kommo_create_task` — a follow-up call on the lead, due tomorrow,
   assigned to that ID.
7. Read back with `kommo_get_lead` — confirm stage, price, and the linked
   contact before reporting done.

## Boundaries

- **A Kommo connection cannot see amoCRM data, and vice versa.** Same API
  family, separate platforms — credentials are not interchangeable. If the
  user's portal is `*.amocrm.ru`, connect the `amocrm` provider instead;
  don't retry `kommo` with amoCRM credentials.
- There are no delete tools for leads, contacts, or companies — close or
  re-stage records instead; hard deletion happens in the Kommo UI.
- `comms` reads call/talk/event history and manages webhook subscriptions;
  it cannot place calls or send chat messages.
- Customer (loyalty) tools work only when the account's customers mode is
  enabled — `kommo_toggle_customers_mode` can switch it, but confirm with
  the user before changing account-level settings.
