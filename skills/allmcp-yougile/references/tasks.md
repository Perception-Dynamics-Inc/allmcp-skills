# YouGile tasks — workflows

Jump to the task that matches the user's request. ✍ marks a step that writes
data — confirm specifics with the user before firing. Enums and ID formats that
cannot be guessed are in the reference tables below.

## Model note — read before acting

- **`column_id` is mandatory on create.** YouGile has no Inbox. If you don't
  have one, call `yougile_list_columns(board_id=<board_id>)` first.
- **No `projectId` filter.** `yougile_list_tasks` does not accept a project ID —
  go via `column_id` (a column within the project's board) or `assigned_to`.
- **`assigned` and `stickers` are full replacements.** Passing `assigned=[]`
  removes all assignees; passing `stickers={"id": "state_id"}` replaces the
  whole sticker map. Merge with the existing values from `yougile_get_task` if
  you want to add without dropping.
- **Deadline is a typed object, not a plain timestamp.** Format:
  `{"deadline": <ms-epoch>, "withTime": bool}`. Add `"startDate": <ms-epoch>`
  and `"startWithTime": bool` for a date range.
- **`include` default differs between list and get.** `yougile_list_tasks`
  defaults to `include="summary"` (compact); `yougile_get_task` defaults to
  `include="full"`. For full task detail after a create, call `yougile_get_task`.

## Task: create a task in a known column

1. `yougile_list_columns(board_id=<board_id>)` → pick the target column → `id`.
2. **✍** `yougile_create_task(title="<title>", column_id=<column_id>)` → `id`.
3. `yougile_get_task(task_id=<id>)` → confirm the task and read its full state.

## Task: find tasks for a user

`yougile_list_tasks(assigned_to=<user_id>)` — returns tasks across all boards.
To narrow to a board, first get the board's columns, then loop or filter
client-side (there is no combined board+assignee filter in one call).

## Task: move a task to a different column (change status)

1. `yougile_list_columns(board_id=<board_id>)` → find the target column → `id`.
2. **✍** `yougile_update_task(task_id=<id>, column_id=<new_column_id>)`.

## Task: complete or archive a task

- Mark done: **✍** `yougile_update_task(task_id=<id>, completed=True)`.
- Archive: **✍** `yougile_update_task(task_id=<id>, archived=True)`.
- Soft-delete: **✍** `yougile_update_task(task_id=<id>, deleted=True)`.
- Restore deleted: **✍** `yougile_update_task(task_id=<id>, deleted=False)`.
- `completed` and `archived` are independent — a task can be completed but not
  archived.

## Task: assign a sticker state to a task

1. `yougile_list_string_stickers(board_id=<board_id>)` → find the sticker by
   name → `sticker_id`.
2. `yougile_get_string_sticker(sticker_id=<id>, include="full")` → read the
   `states` array → pick the matching state → `state_id`.
3. **✍** `yougile_update_task(task_id=<task_id>, stickers={"<sticker_id>": "<state_id>"})`.
   To remove all sticker assignments, pass `stickers={}`.

## Task: manage task chat subscribers

- Read current subscribers:
  `yougile_get_task_chat_subscribers(task_id=<id>)` → `item.content` (list of
  user IDs).
- Replace: **✍** `yougile_update_task_chat_subscribers(task_id=<id>,
  subscribers=[<user_id>, ...])`. Pass `subscribers=[]` to unsubscribe everyone.
- The subscribers list is a full replacement — merge with the existing list from
  the get step if you want to add without removing current subscribers.

## Pagination

`yougile_list_tasks` returns `next_cursor`. Pass it as `offset=<next_cursor>` to
get the next page. Stop when `next_cursor` is `null`.

## Gotchas

- Task IDs are either UUIDs or human codes like `"SAI-515"` — both work in get
  and update.
- Filtering `yougile_list_tasks` by `title` is a substring match, not exact.
- `include="summary"` truncates `description` to 200 chars and replaces subtask/
  sticker arrays with counts. Pass `include="full"` or call `yougile_get_task`
  when you need the full body.
- Chat messages on a task are accessed via the `chat` category using the task ID
  as `chat_id` — task chats and group chats share the same message endpoint.
