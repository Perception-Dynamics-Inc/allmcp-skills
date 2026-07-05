# Bitrix24 communication — workflows

"Communication" = Open Channels: inbound customer chats from connected
messengers (WhatsApp, Telegram, live chat) routed to operator queues. Both
tools are read-only. Phone calls are not open channels — `telephony` category.

## Model note — read before acting

- **Strictly read-only** — no reply/close/transfer/assign/spam-flag, no channel
  create/edit/delete; point such asks to the portal's Contact Center UI.
- **Operator queue membership is not retrievable** in any include mode —
  `QUEUE`/`QUEUE_USERS_FIELDS`/`CONFIG_QUEUE` never appear (the upstream
  option that ships them is never sent). Don't promise a queue answer.
- **No search path to a session.** Nothing maps a customer name or CRM record
  to a session. Default: ask the user for a `session_id`. Fallbacks:
  `bitrix24_list_chats` (`chats` category) may surface the chat ID (open-line
  chats appear there only for participants), or try `bitrix24_list_activities`
  (`crm` category) on the contact/deal — dialogs are logged to CRM.
- `include="summary"` and `"full"` return identical payloads in this category;
  no communication tool has the `efficiency_on` named in the param description.

## Task: read or summarize a customer conversation

1. `bitrix24_get_channel_messages(session_id=4512)`. With only a dialog ID
   from the `chats` category (`"chat1763"`), strip the prefix: `chat_id=1763`.
2. `chat_id` alone resolves to the **newest** session of that chat — an older
   conversation needs its `session_id` (no tool enumerates sessions; ask).
3. The full transcript **likely** arrives regardless of `limit` (`LIMIT` is
   not a documented parameter of this method — per docs, unverified live).
   Never claim "only the latest N are reachable"; be ready to summarize a
   long transcript — but see step 4.
4. A known-active session coming back empty is a retrieval gap, not an empty
   conversation — report "couldn't retrieve the history", never "no messages
   were sent".

This read needs no chat membership — it is the only read path for open-line
dialogs (`bitrix24_get_messages` in `chats` requires membership). `senderid`
values are numeric user IDs — `bitrix24_get_user` (`organization` category).

## Task: list channels / audit a line's settings

1. `bitrix24_list_open_channels()` → match by `LINE_NAME`. Line names are
   portal-configured and the payload may not reveal which messenger feeds a
   line — if no name matches, say so instead of guessing.
2. Filter and read settings client-side (no server-side filter) using the
   table below; a field absent from the payload is not exposed — say so.

Treat this tool as **single-page**: `start` and `next_cursor` are likely inert
(per docs the method paginates via parameters the tool doesn't send, and its
response carries no next/total; unverified live). One call returns up to ~50
lines — enough for virtually all portals; if the user expects more, the tail
is unreachable — say so.

## Reference — line config fields

Flags (`ACTIVE`, `CRM`, `WORKTIME_ENABLE`, …) are `"Y"`/`"N"` strings:

| Field | Values |
|---|---|
| `QUEUE_TYPE` | `evenly` (spread among operators) \| `strictly` (strict queue order) \| `all` (everyone at once) |
| `NO_ANSWER_RULE`, `AUTO_CLOSE_RULE`, `CLOSE_RULE`, `WORKTIME_DAYOFF_RULE` | `none` \| `text` — nothing else; auto-close timeout lives in `AUTO_CLOSE_TIME` |
