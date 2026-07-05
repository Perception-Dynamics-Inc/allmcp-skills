# YouGile projects — workflows

Projects are the top-level workspaces. Boards, columns, and tasks live inside
them. ✍ marks write steps — confirm with the user before firing.

## Model note — read before acting

- **`users` is a `{userId: role}` dict, not a list.** Roles are exactly
  `"admin"`, `"worker"`, or `"observer"` — no other values are accepted.
- **`users` on update is a full replacement.** Merge the existing membership
  with `include="full"` from `yougile_get_project` before writing if you want
  to add one member without removing the rest.
- **No board listing here.** To see a project's boards, use the `boards`
  category: `yougile_list_boards(project_id=<id>)`.
- **Soft-delete preserves data.** `deleted=True` hides the project; `deleted=False`
  restores it. Deleted projects are returned only when `include_deleted=True`.

## Task: list and inspect projects

1. `yougile_list_projects()` → titles + IDs. Filter by name:
   `yougile_list_projects(title="<substring>")`.
2. `yougile_get_project(project_id=<id>)` → full membership and metadata.

## Task: create a project with initial members

**✍** `yougile_create_project(title="<name>", users={"<userId>": "worker", "<userId2>": "admin"})`.

Omit `users` to create an empty project — members can be added later via update.

## Task: add or change a project member's role

1. `yougile_get_project(project_id=<id>, include="full")` → `item.users` dict.
2. Merge the change (add key or update value).
3. **✍** `yougile_update_project(project_id=<id>, users=<merged_dict>)`.

## Task: rename or archive a project

- Rename: **✍** `yougile_update_project(project_id=<id>, title="<new name>")`.
- Archive/delete: **✍** `yougile_update_project(project_id=<id>, deleted=True)`.
- Restore: **✍** `yougile_update_project(project_id=<id>, deleted=False)`.

## Cross-category

- `project_id` → boards: `yougile_list_boards(project_id=<id>)` (`boards` category).
- The full project hierarchy is: project → board → column → task.

## Gotchas

- Valid roles are `"admin"`, `"worker"`, `"observer"` only — passing anything
  else (e.g. `"member"`, `"viewer"`) will fail at the API level.
- `yougile_list_projects` summary returns `id`, `title`, `userCount` — it does
  not include the `users` dict. Call `yougile_get_project(include="full")` to
  read membership before updating it.
