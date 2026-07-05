# Bitrix24 chats — workflows

Three tools: send a message, list recent dialogs, read history. Steps marked
**✍** post a message real people will read — sends cannot be edited or
deleted afterwards, so on a vague request confirm wording and recipient first.

## Model note — read before acting

- **Every message sends under the webhook owner's name**, not necessarily the
  requesting user's. Before first-person text ("I approved it"), check who
  that is via `bitrix24_get_current_user` (organization category) and warn
  the user if identities differ.
- **A 1-on-1 dialog implicitly exists for every user** — any numeric user ID
  works as `dialog_id`, no prior conversation needed.
- **Resolve people via the organization category, never the chat list.**
  `bitrix24_list_chats` is effectively single-page — the ~50 most recent
  dialogs (`next_cursor` always null; `total` usually `-1`, meaning
  "unknown") — so absence there proves nothing and no scan of it is
  exhaustive.
- **Text only, and these three verbs are everything.** No creating chats, no
  participant lists, no reactions, no marking read, no chat-name or
  message-text search.
- **Format with BBCode, never Markdown** (`**bold**` renders literally):
  mention `[USER=123]Name[/USER]` (numeric user ID required),
  `[B]bold[/B]`, `[I]italic[/I]`, `[URL=https://example.com]text[/URL]`.

## Task: DM a colleague ("tell Marina the invoice is approved")

1. `bitrix24_search_users(query="Marina")` (organization category) → user
   `ID`, e.g. `17`. Several matches → ask the user, don't guess. A deal's
   `ASSIGNED_BY_ID` (crm category) is the same kind of ID — usable directly.
2. **✍** `bitrix24_send_message(dialog_id="17", message="Invoice 2214 is approved.")`

## Task: post to a group chat / mention someone

1. `bitrix24_list_chats()` → find the chat by name → use the item's **`id`**
   (already correctly formed: `"chat21"`). **Never pass `chat_id`** — its
   bare integer `21` sent as `"21"` DMs unrelated user 21, irreversibly.
   Chat not listed? There is no chat search — ask the user. (`sgNNN` also
   works: the chat of workgroup NNN, projects category.)
2. **✍** `bitrix24_send_message(dialog_id="chat21",
   message="[USER=87]Aziz[/USER] ticket 4521 needs your review")` — resolve
   the mention's user ID as in the DM recipe.

## Task: catch up on a conversation / find an old message

1. `bitrix24_list_chats()` → a nonzero `counter` is the unread count; take
   the item's `id`.
2. `bitrix24_get_messages(dialog_id="chat21", limit=50)` → newest first;
   **reverse before summarizing** as a transcript.
3. Older history: re-call with `last_id=<lowest message id of the previous
   page>`; a non-existent `last_id` returns an empty page, not an error.
   There is no server-side message search — page and scan client-side,
   sequentially (~2 requests/second portal limit).
4. Messages carry only numeric `author_id` (`0` = system message); names are
   stripped. Resolve via `bitrix24_get_user(b24_user_id=...)` (organization
   category); "my messages" means the webhook owner's ID from
   `bitrix24_get_current_user`.

## Gotchas

- `include="summary"` and `"full"` return identical payloads here — never
  re-call with `"full"` hoping for more fields.
- Attachments are invisible — file data is dropped from messages; a
  file-only message may arrive with empty `text`.
- Open Line (external-customer) dialogs appear in `bitrix24_list_chats`, but
  their history is unreadable here — use `bitrix24_get_channel_messages`
  (communication category).
