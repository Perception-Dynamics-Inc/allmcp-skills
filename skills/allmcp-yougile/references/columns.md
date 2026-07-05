# YouGile columns — workflows

Columns are the swim-lanes inside a board. Every task must live in a column —
there is no inbox or default. ✍ marks write steps.

## Model note — read before acting

- **Column → task is a hard dependency.** `yougile_create_task` requires
  `column_id`. Always resolve a column before creating a task.
- **`board_id` is required to create a column.** Resolve the board first.
- **Color is an integer 1–16** (a palette index, not an RGB value). Omit it to
  let YouGile pick the default.

## Task: list columns on a board (prerequisite for task creation)

`yougile_list_columns(board_id=<board_id>)` → each item has `id`, `title`,
`boardId`. Pass `column_id=<id>` to `yougile_create_task`.

## Task: create a column on a board

1. Resolve `board_id` from `yougile_list_boards` (boards category).
2. **✍** `yougile_create_column(title="<name>", board_id=<board_id>)` → `id`.
3. Use the returned `id` as `column_id` for task creation.

## Task: rename or recolor a column

- Rename: **✍** `yougile_update_column(column_id=<id>, title="<new name>")`.
- Change color: **✍** `yougile_update_column(column_id=<id>, color=<1-16>)`.
- Archive: **✍** `yougile_update_column(column_id=<id>, deleted=True)`.

## Cross-category

- Full hierarchy: project → board → **column** → task.
- After resolving `column_id`, create tasks with `yougile_create_task` (`tasks`
  category). Move a task between columns using `yougile_update_task(column_id=...)`.
