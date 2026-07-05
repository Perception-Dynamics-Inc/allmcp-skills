# Bitrix24 organization — workflows

This category resolves people and departments to the numeric IDs every other
Bitrix24 category consumes (`ASSIGNED_BY_ID` in `crm`, `responsibleId` in
`tasks`, attendees in `calendar`, `user_id_b24` in `telephony`) — and back.
All six tools are read-only: no inviting, editing, or deactivating users; no
creating, renaming, moving, or deleting departments. Org changes need the
Bitrix24 portal UI — say so instead of improvising.

## Model note — read before acting

- **`bitrix24_list_users(query=...)` is an EXACT match on the first-name
  field only.** Last names, full names ("Lena Petrova"), and substrings
  return empty `items` with no error — never treat that as "person doesn't
  exist". For anything except an exact first name, use
  `bitrix24_search_users`. Email is outside Bitrix24's documented search
  scope — treat email-only lookups as unsupported, a miss as inconclusive.
- **`include` is a no-op in this category** — `summary` and `full` return
  identical noise-stripped payloads; never re-call with `include="full"`
  expecting extra fields. The only raw view is `efficiency_on=False` on a get
  tool — also the only way to tell an *empty* field from a *missing* one,
  since noise-strip drops empty values.
- **IDs come back as strings (`"42"`) but go in as ints** — compare loosely
  when chaining.
- Workgroups/projects are **not** departments — they live in the `projects`
  category. Departments here are the formal HR org chart only.

## Task: resolve a person's name to their user ID

1. To find "Lena Petrova": `bitrix24_search_users(query="Petrova")` — one
   distinctive term — confirm via `NAME`/`WORK_POSITION` → user `ID` (`"42"`).
2. Several candidates? Users carry departments as `UF_DEPARTMENT` — an
   **array of int department IDs, never names**. Resolve the IDs with one
   `bitrix24_list_departments()` call; no re-fetch of the user will ever
   yield a department-name field.
3. Zero hits? Fall back to `bitrix24_list_users(query="Lena", active=False)`
   — exact first name — and check `LAST_NAME` client-side. Never put a last
   name or full name in `list_users.query`.

## Task: who is user 42?

`bitrix24_get_user(b24_user_id=42)` — reverse-resolve bare assignee IDs
found on deals, tasks, or calls. `UF_PHONE_INNER` is the internal extension.

## Task: who works in department X / who runs it

1. `bitrix24_list_departments()` → match the department name client-side →
   department `ID`. Sub-units: `bitrix24_list_departments(parent_id=<ID>)`;
   walk upward via each row's `PARENT` (the parent department's ID).
2. Members: `bitrix24_list_users(department_id=<step 1 ID>)`.
3. Head: check `UF_HEAD` (the head's user ID) on the step-1 row. Documented
   department fields are `ID`/`NAME`/`SORT`/`PARENT`; `UF_HEAD` is the only
   head key that can appear — never guess `HEAD`/`head_id` — and it may be
   absent or empty. If absent, re-fetch with
   `bitrix24_get_department(department_id=<ID>, efficiency_on=False)` to
   distinguish "no head assigned" (empty in the raw payload) from "field not
   returned", then `bitrix24_get_user(b24_user_id=<UF_HEAD>)`.

## Task: find former (deactivated) employees

`active=False` REMOVES the active filter — results mix active and
deactivated users. It does not mean "inactive only"; no parameter does.

1. `bitrix24_list_users(active=False)` — page to the end.
2. Keep rows where `ACTIVE` is boolean `false` — not `"N"`. (`IS_ONLINE` in
   the same object *is* `"Y"/"N"` — don't generalize either convention.)
3. Last seen: read `LAST_LOGIN` / `LAST_ACTIVITY_DATE` (ISO-8601) straight
   off each row — no per-user re-fetch needed.

## Task: which Bitrix24 identity am I acting as?

`bitrix24_get_current_user()` returns the webhook creator (often an admin) —
the identity all writes act as, and usually NOT the human you're talking to.
Never present it as "you".

## Pagination & cost

Fixed 50 rows per page — no limit param. Page sequentially: the rate limit
is per portal account and plan-dependent (~2 req/s on standard plans),
shared by every webhook and app on the portal. A full directory scan costs
ceil(N/50) calls — warn the user before scanning a large portal.

## Gotchas

- User lists exclude bots, email-only users, and integration accounts;
  `total` won't match the portal's account count and may drift between pages.
- Personal-field visibility is gated by the webhook's scope tier (`user` /
  `user_basic` / `user_brief`); which fields each tier returns is
  undocumented. A field missing on every user MAY be scope-gated — or just
  empty everywhere. Probe one user with `efficiency_on=False`; if it is
  absent even raw, only re-issuing the webhook with a wider scope can help.
