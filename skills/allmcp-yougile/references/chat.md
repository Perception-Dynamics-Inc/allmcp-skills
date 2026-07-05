# YouGile chat — workflows

Covers group chats and the messages endpoint, which works for both group chats
and task-embedded chats. ✍ marks write steps — confirm before firing.

## Model note — read before acting

- **`chat_id` accepts a group-chat ID or a task ID.** Every task has an
  implicit chat. To send or read messages on a task's thread, pass the task ID
  as `chat_id` — no separate "task chat" resource exists.
- **PUT (update) only accepts `text` and `deleted`.** The fields `text_html`
  and `label` are POST-only (set at send time). Editing these after the fact is
  not supported — editing a message replaces only the plain `text`.
- **Group chat user roles are `"admin"` or `"member"` only** (different from
  project roles). Passing `"worker"` or `"observer"` fails.
- **`users` on group chat update is a full replacement.** Read the current
  membership from `yougile_get_group_chat(include="full")` before updating to
  avoid dropping existing members.

## Task: send a message to a task's chat thread

`yougile_send_chat_message(chat_id=<task_id>, text="<message>")` → `id`.
No prior lookup needed — every task has a chat. The task ID IS the chat ID.

## Task: read recent messages in a task or group chat

`yougile_list_chat_messages(chat_id=<task_id_or_chat_id>)` → messages in
reverse-chronological or arrival order (use `offset` to paginate older messages).
Filter by author: `from_user_id=<user_id>`. Filter by text: `text=<substring>`.

## Task: create a group chat and add members

1. **✍** `yougile_create_group_chat(title="<name>", users={"<userId>": "member",
   "<userId2>": "admin"})` → `id`.
2. Confirm with `yougile_get_group_chat(chat_id=<id>)`.

## Task: add a member to an existing group chat

1. `yougile_get_group_chat(chat_id=<id>, include="full")` → `item.users` dict.
2. Merge the new user in: `users["<new_user_id>"] = "member"`.
3. **✍** `yougile_update_group_chat(chat_id=<id>, users=<merged_dict>)`.

## Task: edit or delete a sent message

- Edit text: **✍** `yougile_update_chat_message(chat_id=<id>, message_id=<msg_id>,
  text="<new text>")`.
- Soft-delete: **✍** `yougile_update_chat_message(chat_id=<id>, message_id=<msg_id>,
  deleted=True)`.
- `text_html` cannot be updated after send — only `text` is accepted by PUT.

## Pagination

`yougile_list_chat_messages` and `yougile_list_group_chats` both return
`next_cursor`. Pass as `offset=<next_cursor>` for the next page.

## Gotchas

- `yougile_list_chat_messages` summary truncates `text` to 500 chars. If message
  bodies are needed in full, pass `include="full"` or call `yougile_get_chat_message`.
- A task ID used as `chat_id` bypasses the group-chat CRUD tools — the task's
  chat cannot be renamed or deleted independently.
- Subscriber management for task chats (who gets notified) is done via
  `yougile_update_task_chat_subscribers` in the `tasks` category, not here.
