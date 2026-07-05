# Bitrix24 CRM — workflows

Jump to the task matching the user's request. The reference tables at the
bottom hold the values you cannot guess (`stage_id`, `status_id`, `source_id`,
`type_id`). Steps marked **✍** create or change data in the user's CRM — if
the request was vague, confirm specifics before firing a ✍ step.

## Model note — read before acting

- **Writes can silently no-op.** Bitrix24 ignores invalid field values on
  create and update and still reports success — `success=True` or a returned
  `id` proves nothing about individual fields. After any write that sets a
  directory-backed field (`stage_id`, `status_id`, `source_id`, `industry`),
  re-read the record with the matching `get_*` tool and confirm the field
  actually holds your value before reporting success to the user.
- **This category is create + read, not full edit.** No update tool exists for
  contacts, companies, leads, or activities; deals expose only
  `bitrix24_update_deal_stage`; nothing can be deleted; estimates are
  read-only. Don't promise to rename a contact, change a phone number, edit a
  deal amount, remove a record, or attach product line items to a deal (the
  product catalog lives in the `commerce` category; no bridge tool exists).
- **Lead vs deal.** A lead is an unqualified prospect; a deal is an active
  opportunity. Default new inbound interest to a lead — Bitrix24's flow
  qualifies a lead into contact+company+deal later. Create a deal directly
  only when the user says it's qualified or names an existing customer.
- **User IDs are numeric** — `responsible_id` and `ASSIGNED_BY_ID` take
  Bitrix24 user IDs; resolve names via `bitrix24_list_users` in the
  `organization` category.

## Task: find a contact or company by name

Contact `query` matches the **first-name field only**. `query="Sarah Chen"`
and `query="Chen"` both return nothing even when Sarah Chen exists.

1. `bitrix24_list_contacts(query="Sarah")` → scan `LAST_NAME` in the results
   to pick the right person → `ID`.
2. Companies match on title substring: `bitrix24_list_companies(query="Acme")`.
3. Empty `items` is not proof of absence when you only know a surname — say so
   instead of declaring "not in CRM". You cannot search by phone or email.

## Task: log a new inbound enquiry

A fresh, unqualified enquiry is a lead — not a deal.

1. **✍** `bitrix24_create_lead(title="Website enquiry — pricing", name="Jane",
   last_name="Doe", phone="+15551234567", email="jane@acme.com",
   source_id="WEB")` → lead `id`.
2. `bitrix24_get_lead(lead_id=<step 1>)` → confirm `SOURCE_ID` reads `WEB` —
   an unknown source value is silently dropped.

`source_id` values: see the Lead sources table.

## Task: create a deal for a customer

1. `bitrix24_list_companies(query="Acme")` → company `ID`. If none:
   **✍** `bitrix24_create_company(title="Acme Corp")` → `id`. Pass `industry`
   only when copying the value from an existing company, and confirm it stuck
   via `bitrix24_get_company` — free-text values are silently dropped.
2. `bitrix24_list_contacts(query="Sarah")` (first name only) → contact `ID`.
   If none: **✍** `bitrix24_create_contact(name="Sarah", last_name="Chen",
   email="sarah@acme.com", company_id=<step 1>)` → `id`.
3. **✍** `bitrix24_create_deal(title="Acme — annual licence",
   company_id=<step 1>, contact_id=<step 2>, opportunity=12000,
   currency_id="USD")` → deal `id`.

Omit `stage_id` to land in the pipeline's first stage — the safe default. If
you do set one, add: 4. `bitrix24_get_deal(deal_id=<step 3>)` → confirm
`STAGE_ID`; an invalid stage silently lands the deal in the first stage
instead. One contact per deal — more cannot be attached here.

## Task: move a deal through the pipeline / mark won or lost

`stage_id` values are pipeline-specific and no tool enumerates them, so never
guess — copy the convention from the deal itself:

1. `bitrix24_get_deal(deal_id=123)` → read current `STAGE_ID`. Bare (`NEW`) =
   default pipeline, use the Deal stages table below. Prefixed (`C2:NEW`) =
   custom pipeline, keep the same `C{n}:` prefix on the new stage.
2. **✍** `bitrix24_update_deal_stage(deal_id=123, stage_id="C2:WON")`.
3. `bitrix24_get_deal(deal_id=123)` → confirm `STAGE_ID` equals what you set.
   If unchanged, your value was invalid for this pipeline: list other deals
   from the same pipeline and reuse a `STAGE_ID` you actually observe.

These tools cannot move a deal between pipelines (only the Bitrix24 UI can).
To list deals in a stage: `bitrix24_list_deals(stage_id="C2:WON")` — exact
match, prefix included.

## Task: qualify a lead into a customer + deal

There is **no conversion tool** and no way to update a lead. Recreate by hand:

1. `bitrix24_get_lead(lead_id=45)` → read name, phone, email, company info.
2. **✍** Create company → contact (with `company_id`) → deal, per the recipe
   above, copying the lead's data.
3. Tell the user the lead itself stays open — its `STATUS_ID` cannot be set to
   `CONVERTED` from here.

## Task: review or log activity on a record

Activities bind to exactly one CRM record (`owner_type_id` + `owner_id`). To
cover a customer fully, check the company, its contacts, and their deals
separately.

**Read history:** `bitrix24_list_activities(owner_type_id=2, owner_id=123)` →
activities on deal 123. There is no type or date filter — filter client-side
on `TYPE_ID` using the Activity types table (emails are `4`, not `3`). The
summary projection's only date field is `DEADLINE` — for "when" questions pass
`include="full"` and use whatever date fields the payload actually exposes.

**Log an activity** — activities are permanent (no complete/edit/delete
afterwards), so confirm before writing:

1. **✍** `bitrix24_create_activity(owner_type_id=2, owner_id=123,
   subject="Discovery call", type_id=2, communication_value="+15551234567",
   description="Discussed pricing; follow-up next week")`.

Always pass `type_id` explicitly. There is no "note" type — to preserve a
plain remark (e.g. a lost reason), log `type_id=5` (Action) rather than a
phantom call or meeting. For an actionable to-do, use `bitrix24_create_task`
in the `tasks` category instead of `type_id=3`.

## Task: look up a quote/estimate

Estimates are read-only: `bitrix24_list_estimates(query="Acme")` →
`bitrix24_get_estimate(estimate_id=7)`. Quote line items are unreachable, and
no quote can be created or edited — don't offer to. Quotes reference deals,
contacts, and companies by numeric ID — feed any ID you find into the `get_*`
tools above.

## Rate limit

Page `list_*` calls sequentially, never in parallel — the portal allows
around 2 requests/second, shared across the whole portal.

## Gotchas

- Phone/email round-trip asymmetry: you *pass* plain strings to `create_*`
  tools, but `include="full"` payloads return arrays of `{VALUE, VALUE_TYPE}`
  — read `PHONE[0].VALUE`. `include="summary"` pre-flattens to the first
  value.
- `total` on list responses can drift between pages while data changes; treat
  it as an estimate during long scans.

## Reference — IDs & enums

Stage/status/source directories are **portal-configurable**; tables below show
system defaults. When in doubt, read an existing record and reuse the exact
value you observe.

**Deal stages** (default pipeline `stage_id`):

| `stage_id` | Meaning |
|---|---|
| `NEW` | New |
| `PREPARATION` | Document preparation |
| `EXECUTING` | In progress |
| `WON` | Won — terminal |
| `LOSE` | Lost — terminal (**not** `LOST`) |

**Lead statuses** (`status_id`, list-filter only — leads cannot be updated
here): `NEW`, `IN_PROCESS`, `PROCESSED`, `JUNK`, `CONVERTED` (set only by
Bitrix24's own conversion flow).

**Lead sources** (`source_id`, common defaults): `CALL`, `EMAIL`, `WEB`,
`ADVERTISING`, `PARTNER`, `RECOMMENDATION`, `WEBFORM`, `CALLBACK`, `OTHER`.

**Activity types** (`type_id`): `1`=Meeting, `2`=Call, `3`=Task (prefer the
`tasks` category), `4`=E-mail, `5`=Action (generic note).
