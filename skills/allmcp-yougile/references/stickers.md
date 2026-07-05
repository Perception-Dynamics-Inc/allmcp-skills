# YouGile stickers — workflows

YouGile has two distinct sticker families with different state management.
✍ marks write steps. The assignment ID flow (sticker → state → task) is the
main cross-category dependency.

## Model note — read before acting

- **Two separate families — different endpoints, different semantics.**
  - **String stickers** — custom labels with colored states (e.g. "Priority"
    with states "Low", "Medium", "High"). States are managed via nested
    `create/update_string_sticker_state` tools.
  - **Sprint stickers** — time-bounded sprint windows (`begin`/`end` as ms-epoch
    timestamps). States are passed inline via the parent body's `states` array.
- **Assigning a sticker to a task requires two IDs**: the sticker ID and the
  state ID. Use `yougile_get_string_sticker(include="full")` to read the states
  array and find state IDs.
- **`begin`/`end` on sprint states are millisecond-epoch timestamps** (Unix ms,
  not seconds). Multiply seconds by 1000. A sprint with no `begin` or `end` is
  open-ended.
- **Colors are integers 1–16** (a palette index). Omit to let YouGile pick the
  default.
- **No tool to list states directly.** Read them from the parent sticker via
  `yougile_get_string_sticker(include="full")` → `item.states`.

## Task: assign a string sticker state to a task

1. `yougile_list_string_stickers(board_id=<board_id>)` → find the sticker by
   name → `sticker_id`.
2. `yougile_get_string_sticker(sticker_id=<sticker_id>, include="full")` →
   `item.states` array → find the matching state by name → `state_id`.
3. **✍** `yougile_update_task(task_id=<task_id>, stickers={"<sticker_id>": "<state_id>"})`
   (tasks category). Pass the full current sticker map to avoid clearing other
   sticker assignments — merge with the existing `stickers` field from
   `yougile_get_task(include="full")`.

## Task: create a string sticker with initial states

**✍** `yougile_create_string_sticker(name="Priority", states=[{"name": "Low",
"color": 3}, {"name": "Medium", "color": 8}, {"name": "High", "color": 1}])` → `id`.

States can also be added after creation with `yougile_create_string_sticker_state`.

## Task: add or rename a state on an existing string sticker

- Add: **✍** `yougile_create_string_sticker_state(sticker_id=<id>, name="<state name>",
  color=<1-16>)` → `state_id`.
- Rename: **✍** `yougile_update_string_sticker_state(sticker_id=<id>, state_id=<id>,
  name="<new name>")`.
- Delete state: **✍** `yougile_update_string_sticker_state(sticker_id=<id>,
  state_id=<id>, deleted=True)`.

## Task: create a sprint sticker with sprint windows

**✍** `yougile_create_sprint_sticker(name="Q3 2026", states=[{"name": "Sprint 1",
"begin": 1751328000000, "end": 1752537600000}])` → `id`.

`begin` and `end` are ms-epoch. To update sprints, replace the whole `states`
list via `yougile_update_sprint_sticker(sticker_id=<id>, states=[...])`.

## Cross-category

- Stickers visible on a board: `yougile_list_string_stickers(board_id=<board_id>)`
  or `yougile_list_sprint_stickers(board_id=<board_id>)`.
- Assign sticker state to a task: `yougile_update_task(stickers=...)` (tasks
  category). Remove all assignments: `stickers={}`.
- The `stickers` object on a board (`yougile_create_board`, `yougile_update_board`)
  is an opaque attachment map — read it from an existing board before mutating.

## Gotchas

- String and sprint stickers are unrelated resource types — don't mix their IDs
  or endpoints.
- `yougile_list_string_stickers` summary returns `id`, `name`, `icon`,
  `stateCount`. To see individual state IDs and names, call `yougile_get_string_sticker(include="full")`.
- Sprint sticker `states` on update is a full replacement — not a merge.
  Read the existing states before updating if you want to preserve current sprints.
