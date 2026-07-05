# Bitrix24 tasks — workflows

Jump to the recipe matching the request; status semantics are at the bottom.
Steps marked **✍** change data — confirm vague requests before firing one.

## Model note — read before acting

- **No reassign, move, or delete** (assignee and project are create-only); no
  creating subtasks, comments, checklists, time entries, attachments, extra
  participants, or CRM bindings (reference CRM records in the description text
  instead). Don't promise these; the last recipe covers the disclosure-first
  workaround.
- **Resolve names to numeric IDs in the right category — never mine task
  payloads for them** (payloads carry only IDs like `responsibleId`, and the
  full field set is portal-dependent). People: `bitrix24_search_users`
  (fulltext) or `bitrix24_list_users` (exact name filter) in the
  **organization** category; "assign to me" → `bitrix24_get_current_user`.
  Projects/workgroups: `bitrix24_list_workgroups` in the **projects**
  category. Never guess an ID.
- **`bitrix24_update_task` reports success without checking the result.**
  After any write whose outcome you'll report (status, deadline), read back
  with `bitrix24_get_task` and report the stored values.

## Task: create and assign a task

1. `bitrix24_search_users(query="Marina")` (organization) → user `id` 42.
2. Project mentioned? `bitrix24_list_workgroups(query="Brightline")`
   (projects) → group `id` 17. Omit `group_id` for a standalone task.
3. **✍** `bitrix24_create_task(title="Prepare renewal deck",
   responsible_id=42, group_id=17, deadline="2026-06-12T18:00:00",
   priority=2)` → task `id`.
4. Sent a deadline? `bitrix24_get_task(task_id=<step 3>)` and report the
   stored `deadline` — how a bare ISO datetime's timezone is interpreted is
   unverified, so report what stuck, not what you sent.

## Task: what's overdue / on someone's plate

1. Resolve the person to a user ID (organization category) → e.g. 42.
2. `bitrix24_list_tasks(responsible_id=42, status="-1")` → overdue tasks,
   computed server-side (`"-3"` deadline ≤ tomorrow, `"-2"` unviewed by the
   assignee). Don't paginate-and-date-filter client-side — the meta-filters
   do it in one call.

## Task: list a project's tasks

1. `bitrix24_list_workgroups(query="Apollo")` (projects) → group `id` 17.
2. `bitrix24_list_tasks(group_id=17)`.

## Task: mark a task done, postpone it, or push a deadline

1. Only have a title? `bitrix24_list_tasks(query="renewal deck")` → task `id`.
2. **✍** `bitrix24_update_task(task_id=4821, status=5)` (postpone: `status=6`;
   new deadline: `deadline="2026-06-15T09:00:00"`).
3. `bitrix24_get_task(task_id=4821)` → confirm the stored status/deadline.
   Status `"4"` (Awaiting control) means the portal routes completions through
   the creator's review — report the task as awaiting approval, not closed;
   approving is not possible through this connector.

## Task: reassign or move a task (unsupported — disclose, then recreate)

The only workaround is recreating the task under the new owner — it is lossy:

1. Disclose first: the copy carries ONLY title, description, project,
   deadline, priority. Comments, checklists, attachments, time tracking,
   participants, CRM links, and subtasks do NOT carry over. Get a go-ahead.
2. `bitrix24_get_task(task_id=4830)` → copy the fields.
3. **✍** `bitrix24_create_task(title=<same>, responsible_id=<new owner>,
   description=<same>, group_id=<same>, deadline=<same>, priority=<same>)`.
4. **✍** `bitrix24_update_task(task_id=4830, status=6)` — defer the original
   so it doesn't linger as an open duplicate (or leave it, per the user).
5. `bitrix24_get_task(task_id=4830)` → confirm the stored status before
   reporting the handoff done.

## Gotchas

- `query` is a substring match on the title — "report" matches "Q3 report
  draft". It does not search descriptions or project names.
- All requests share one per-portal rate limit — filter server-side rather
  than walking every page.

## Reference — status semantics

`status` is a string in read payloads but an int on update. The
`"-1"`/`"-2"`/`"-3"` meta-values are list-filter-only — computed server-side,
never stored on a task. For status `"4"` semantics see the mark-done recipe.
