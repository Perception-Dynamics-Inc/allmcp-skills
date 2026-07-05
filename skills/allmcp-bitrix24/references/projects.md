# Bitrix24 projects — workflows

**Read-only** — creating, renaming, archiving, and membership changes all need
the Bitrix24 UI. A "project" is just a workgroup; the main job here is
resolving a name to the numeric `group_id` other categories consume. Steps
marked **✍** write data — if the request was vague, confirm specifics first.

## Model note — read before acting

- **The two tools disagree on key casing.** List rows are camelCase (`id` may
  be a string — use its numeric value as `group_id`); get payloads are
  UPPERCASE (`ID`, `NAME`, `OWNER_ID`, `CLOSED`). Take `item["id"]` from
  list, read UPPERCASE keys from get; never pattern-match across the two.
- **Empty results don't prove absence.** The list is permission-filtered per
  webhook user and "not found" also fires on no-access — report "not visible
  to this connection", never "this project doesn't exist".
- **The get payload is all there is.** Select-only extras (`OWNER_DATA`,
  `COUNTERS`, `EFFICIENCY`, member detail like `LIST_OF_MEMBERS`) can never
  appear, and `efficiency_on=False` never adds fields — don't re-fetch raw
  for more.

## Task: find a project and get its ID

1. `bitrix24_list_workgroups(query="Redesign")` — substring match: "site" matches "Website Redesign".
2. Rows may carry little beyond `id`. Confirm the real match — and any
   duplicate-name claim — with `bitrix24_get_workgroup(group_id=<item["id"]>)`
   per candidate, sequentially (~2 req/s shared limit); compare exact `NAME`.

## Task: list or create tasks in a project

1. Resolve the project → `id` 17 (recipe above).
2. `bitrix24_list_tasks(group_id=17)` (**tasks** category); overdue: add `status="-1"`.
3. New task: **✍** `bitrix24_create_task(title="Kickoff deck",
   responsible_id=42, group_id=17)` — names→IDs first via
   `bitrix24_search_users(query="Dmitry")` (**organization** category) — then
   `bitrix24_get_task(task_id=<id from create>)` to confirm the stored `groupId`.

## Task: inspect a project — owner, archived or active, deadlines

1. `bitrix24_get_workgroup(group_id=17)` → `OWNER_ID`, flags below, and `PROJECT_DATE_START/FINISH` (projects only).
2. `OWNER_ID` and member ID arrays such as `MEMBERS`/`MODERATOR_MEMBERS`
   (when present) hold numeric user IDs, never names — resolve each via
   `bitrix24_get_user(b24_user_id=<OWNER_ID>)` in the **organization**
   category; ID→name is never a dead end.

## Reference — flags, types & pagination

Get-payload flags are `"Y"`/`"N"` strings: `CLOSED` (`"Y"` = archived), `ACTIVE`,
`PROJECT`, `OPENED` (`"Y"` = anyone joins without approval). Type (`type` on
list, `TYPE` on get): `group` | `project` | `scrum` | `collab`. Lists return
50/page — feed `next_cursor` back as `start`; stop when null.
