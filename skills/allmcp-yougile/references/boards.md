# YouGile boards — workflows

Boards are Kanban surfaces that belong to a project. Columns and tasks live
inside boards. ✍ marks write steps.

## Model note — read before acting

- **`project_id` is required to create a board.** Always resolve the project
  first via the `projects` category.
- **The `stickers` field on board create/update is opaque.** Don't invent its
  structure — call `yougile_list_string_stickers(board_id=<id>)` on an existing
  board to read the current map, then mirror the shape in your write.
- **No column listing from this category.** To see a board's columns, use
  `yougile_list_columns(board_id=<id>)` in the `columns` category.

## Task: discover boards in a project

1. `yougile_list_projects()` (projects category) → `project_id`.
2. `yougile_list_boards(project_id=<project_id>)` → boards for that project.
3. `yougile_get_board(board_id=<id>)` for full detail.

## Task: create a board in a project

1. Resolve `project_id` from `yougile_list_projects()`.
2. **✍** `yougile_create_board(title="<name>", project_id=<project_id>)` → `id`.

## Task: rename or move a board

- Rename: **✍** `yougile_update_board(board_id=<id>, title="<new name>")`.
- Move to another project: **✍** `yougile_update_board(board_id=<id>, project_id=<new_project_id>)`.
- Archive: **✍** `yougile_update_board(board_id=<id>, deleted=True)`.

## Cross-category

- `board_id` → columns: `yougile_list_columns(board_id=<id>)` (`columns` category).
- `board_id` → stickers: `yougile_list_string_stickers(board_id=<id>)` (`stickers` category).
- Full hierarchy: project → **board** → column → task.

## Gotchas

- `yougile_list_boards` summary returns `id`, `title`, `projectId` only.
  The full payload (returned by `yougile_get_board(include="full")`) includes the
  `stickers` attachment map.
- Soft-deleted boards are hidden by default; pass `include_deleted=True` to list
  them, then restore with `yougile_update_board(board_id=<id>, deleted=False)`.
