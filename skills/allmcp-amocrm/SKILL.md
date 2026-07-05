---
name: allmcp-amocrm
description: Work amoCRM (*.amocrm.ru) — leads, contacts, companies, pipelines, tasks, the unsorted inbox — through the AllMCP hub. Use when the user asks to read or change anything in their amoCRM account via AllMCP. NOT for Kommo accounts on *.kommo.com — that is the allmcp-kommo skill; amocrm.ru and kommo.com are separate platforms and credentials don't cross.
---

# amoCRM via AllMCP

amoCRM is the sales-pipeline CRM dominant in RU/CIS markets. Through AllMCP
your agent gets ~91 namespaced tools (`amocrm_list_leads`,
`amocrm_update_lead`, …) across 12 categories. This one file is the whole
playbook — there is no `references/` folder. Platform mechanics (endpoint,
hidden tools, quota, error loop) live in the `allmcp` meta-skill — install
it alongside this one.

## Connect

- Provider key: `amocrm` (exactly this). If the user's URL ends in
  `.kommo.com`, stop — that's the `kommo` provider, not this one.
- **OAuth2 (default):** call `connect_provider(provider_key="amocrm")` with
  nothing else. The response is `action_required` with a consent URL — hand
  it to the user and stop. They sign in to their amoCRM account and approve;
  their tenant subdomain is captured automatically from the callback, and
  AllMCP refreshes tokens from then on. Never ask an OAuth user for client
  IDs, secrets, or tokens — there is nothing to paste.
- **Long-lived token (fallback):** if the user generated one in their portal
  (Settings → Integrations → Long-lived token), pass
  `extra_fields={"subdomain": "shop123", "access_token": "<token>"}` —
  subdomain is the bare label, without `.amocrm.ru`.

## Model note — read before acting

- **Four entities carry the sale, and they don't auto-link.** A *lead* is
  the deal in the pipeline; a *contact* is a person; a *company* is an org;
  a *customer* is a separate loyalty/repeat-purchase record (module may be
  switched off in the portal). Creating a lead attaches nobody — link
  explicitly (`amocrm_link_lead`) or create atomically
  (`amocrm_create_lead_complex`).
- **Never guess a `status_id`.** Pipeline stages are portal-configured with
  arbitrary numeric IDs; every account differs. Read
  `amocrm_list_pipelines` / `amocrm_list_statuses` first, and note a
  status_id is only valid inside its own pipeline.
- **Inbound leads may not be leads yet.** Form fills, chats, and calls land
  in the *unsorted* inbox as records keyed by a **string `uid`** — they
  become real leads only when accepted.
- **Everything else is a numeric ID** — leads, contacts, users, pipelines,
  even custom fields (addressed by field ID, not name; discover them via
  `amocrm_list_custom_fields`). Resolve people with `amocrm_list_users`.
- **The API throttles at ~7 requests/second** and repeated hammering after
  429s can escalate to an IP-level block — pace bulk jobs, don't
  parallelize aggressively.
- Cyrillic names and field values are normal in these tenants — pass them
  through as-is, never transliterate.

## Categories

Call `describe_category("amocrm", <category>)` before first use of a
category — it enables the tools and returns the provider team's workflow
guide. Only `leads`, `contacts`, `companies` are pre-unlocked on connect.

| Category | Covers |
|---|---|
| `leads` | Leads — sales pipeline opportunities |
| `contacts` | Contacts — individuals (people) |
| `companies` | Companies — organisations linked to leads/contacts |
| `customers` | Customers (loyalty) — repeat-purchase entities, segments, bonus points |
| `tasks` | Tasks — assigned to-dos with due dates |
| `notes` | Notes — free-form notes attached to leads/contacts/customers |
| `pipelines` | Pipelines and stages — sales-process configuration |
| `catalogs` | Catalogs — custom record types (products/SKUs) and elements |
| `custom_fields` | Custom field definitions and metadata |
| `users` | Users and roles — workspace membership |
| `unsorted` | Unsorted incoming-inbox leads (forms, chats, calls) |
| `comms` | Communications — calls, talks, events, webhooks |

## Task: land a new inbound lead with the contact attached

✍ marks a step that writes to the CRM.

1. `amocrm_list_pipelines()` → find the target pipeline, e.g. `"Продажи"`
   id `7423598`, and its first stage's `status_id` `60123454` from the
   embedded statuses.
2. ✍ `amocrm_create_contact(name="Марина Иванова", phone="+79123456789",
   email="marina@example.ru")` → contact id `9245771`.
3. ✍ `amocrm_create_lead(name="Ремонт офиса", price=450000,
   pipeline_id=7423598, status_id=60123454)` → lead id `31337001`.
4. ✍ `amocrm_link_lead(lead_id=31337001, to_entity_id=9245771,
   to_entity_type="contacts")` — without this the lead sits orphaned.
5. ✍ Optional follow-up: `amocrm_create_task(text="Перезвонить",
   complete_till=<epoch seconds>, entity_id=31337001, entity_type="leads")`
   — `complete_till` is required; amoCRM rejects deadline-less tasks.
6. Read back: `amocrm_get_lead(lead_id=31337001)` — the default embed
   includes contacts; confirm `9245771` appears before reporting done.

Shortcut when the contact is name-only: `amocrm_create_lead_complex(...)`
creates lead + contact + company in one atomic call, but returns only the
lead id and can't set the contact's phone/email — use the explicit path
above when you have real contact details.

## Task: move a lead through the pipeline correctly

1. `amocrm_get_lead(lead_id=31337001)` → current `pipeline_id` and
   `status_id`.
2. `amocrm_list_statuses(pipeline_id=7423598)` → the account's actual
   stages; find the target, e.g. `"Переговоры"` → `status_id=60123458`.
   Won/lost are statuses in this list too — closing a deal is also a move.
3. ✍ `amocrm_update_lead(lead_id=31337001, status_id=60123458)`. Moving
   across pipelines? Pass **both** `pipeline_id` and `status_id`, and the
   status must belong to the new pipeline — a mismatched pair is the most
   common 400 here.
4. Read back `amocrm_get_lead(lead_id=31337001)` and confirm the
   `status_id` changed before reporting the move.

## Task: clear the unsorted inbox

1. `amocrm_unsorted_summary()` for counts, then `amocrm_list_unsorted()` →
   rows keyed by string uid, e.g. `uid="fd8062ba33cf9…"`, with the source
   (form/chat/call) and draft lead data embedded.
2. `amocrm_get_unsorted(uid="fd8062ba33cf9…")` to inspect before deciding.
3. ✍ Accept the real ones: `amocrm_accept_unsorted(uid="fd8062ba33cf9…",
   status_id=60123454)` — converts it into a lead in that stage (status_id
   from a pipelines read, never guessed). ✍ Reject spam with
   `amocrm_decline_unsorted(uid=...)`. A chat that belongs to an existing
   deal: ✍ `amocrm_link_unsorted(uid=..., entity_id=..., entity_type="leads",
   user_id=...)` — link targets only leads or customers, never contacts.
4. Accept doesn't return the new lead id — read back with
   `amocrm_list_leads()` (newest first) and confirm the uid is gone from
   `amocrm_list_unsorted()`.

## Boundaries

- **No deletes of core records.** Leads, contacts, companies, customers,
  tasks, and catalog elements cannot be deleted through this connector —
  only notes, pipelines/statuses, custom fields, and webhook subscriptions
  have delete tools. Declining an unsorted record is the one way to make an
  incoming lead disappear.
- **Comms is logging and reading, not messaging.** `amocrm_create_call_note`
  records that a call happened — nothing here dials, sends SMS, or replies
  in chats; talks can be listed and closed, not answered.
- **Webhook tools manage subscriptions only** — the destination URL must be
  an endpoint the user operates; AllMCP does not receive events for you.
- **Users and roles are read-only** — list and inspect, no inviting or
  editing members.
- **No file uploads or attachments** — the amoCRM Files API is not covered.
