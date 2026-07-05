---
name: allmcp-bitrix24
description: Work Bitrix24 — CRM, tasks, calendar, chats, drive, telephony — through the AllMCP hub. Use when the user asks to read or change anything in their Bitrix24 portal (contacts, companies, leads, deals, tasks, events, messages, files, call history) via AllMCP. NOT for calling Bitrix24's REST API directly with your own code, or portal administration (installing apps, editing user rights).
---

# Bitrix24 via AllMCP

Bitrix24 is a CRM and business platform. Through AllMCP your agent gets
namespaced tools (`bitrix24_list_deals`, `bitrix24_create_task`, …) organized
into categories. This file is orientation and routing; the real playbooks are
in `references/` — read the matching one **before** chaining tools in a
category you haven't used this session.

Platform mechanics (endpoint, connecting, hidden tools, quota) live in the
`allmcp` skill — install it alongside this one.

## Connect

- Provider key: `bitrix24` (exactly this, snake_case).
- Credential: a Bitrix24 **inbound webhook URL**, passed as the single
  `api_key` field of `connect_provider`. The user creates it in their portal:
  **Apps → Developer resources → Other → Inbound webhook**, grants scopes
  (CRM, Tasks, Calendar, …), and copies the URL — it looks like
  `https://your-portal.bitrix24.ru/rest/1/abc123secret/`.
- The webhook URL is a live credential — never echo it back in full, never
  log it, and if the user pastes it somewhere public, tell them to revoke it
  in Bitrix24.
- Tools only cover scopes the webhook was granted. A working connection that
  errors on one category usually means a missing scope, not a broken portal —
  have the user re-create the webhook with the scope added.

## The two rules that prevent most failures

1. **Everything is a numeric ID.** Contacts, deals, users, tasks, folders —
   all referenced by integer IDs. Never pass a name where an ID is expected;
   resolve people via `bitrix24_list_users` (`organization` category) and
   look up records with the matching `list_*` tool first.
2. **The portal throttles at roughly 2 requests/second** per webhook and
   queues the excess. Large batch jobs run slower than you expect — don't
   parallelize aggressively, and warn the user before hundred-record sweeps.

## Categories → playbooks

Read the reference file before working in a category; each one carries the
unguessable enums (deal stages like `C2:WON`, activity types), the ID-flow
between tools, and the boundaries.

| Category | Read | Covers |
|---|---|---|
| `crm` | [references/crm.md](references/crm.md) | Contacts, companies, leads, deals & stages, activities, quotes |
| `tasks` | [references/tasks.md](references/tasks.md) | Create/assign tasks, deadlines, statuses, overdue lists |
| `calendar` | [references/calendar.md](references/calendar.md) | Agendas, availability, booking events |
| `chats` | [references/chats.md](references/chats.md) | DMs, chat history, mentions |
| `collaboration` | [references/collaboration.md](references/collaboration.md) | News-feed posts |
| `communication` | [references/communication.md](references/communication.md) | Open Channels transcripts (read-only) |
| `telephony` | [references/telephony.md](references/telephony.md) | Call history & stats, recording links — **no dialing** |
| `organization` | [references/organization.md](references/organization.md) | Users & org chart — the name→ID resolver for everything else |
| `projects` | [references/projects.md](references/projects.md) | Workgroups/projects; IDs feed the task tools |
| `drive` | [references/drive.md](references/drive.md) | Drives, folders, file metadata, download links (read-only) |
| `lists` | [references/lists.md](references/lists.md) | Custom lists/registries rows (read-only) |
| `commerce` | [references/commerce.md](references/commerce.md) | Store products, orders, payments (read-only) |
| `content` | [references/content.md](references/content.md) | Sites, landing pages, generated documents (read-only) |
| `automation` | [references/automation.md](references/automation.md) | Start Business Process workflows, list running instances |
| `time_tracking` | [references/time_tracking.md](references/time_tracking.md) | Clock in/out, workday status |

Thin categories (`booking`, `storage`, `surveys`) have no playbook — their
tool descriptions carry everything needed; discover them with
`describe_category("bitrix24", ...)`.

## Boundaries worth knowing up front

- Several categories are deliberately **read-only** (commerce, drive, lists,
  content, communication) — don't promise writes there.
- Telephony reads history and recordings; it cannot place calls.
- Time tracking is clock in/out only — there is no per-task worklog surface.
