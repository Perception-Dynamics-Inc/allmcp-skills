# Bitrix24 collaboration (news feed) — workflows

This category covers the news feed only (workgroups live in `projects`).
**✍** = irreversible write: confirm message and audience first if vague.

## Model note — read before acting

- **Posts are write-once.** No edit/delete/comment/get-by-ID tools exist — a
  bad post can only be fixed in the Bitrix24 UI.
- **Omitting `dest` posts to the ENTIRE COMPANY** (provider default `UA`) —
  never omit it as a neutral fallback; resolve the audience or ask.
- **One `dest` token per post** — two audiences means two separate ✍ posts.
- **Documented post rows carry no recipient field** — never rely on the feed
  to reveal a group ID or a past post's scope; resolve audiences in their
  home category (or ask the user) instead.
- `include="summary"` and `include="full"` return identical rows here; ignore
  the param's pointer to a `get_*` tool — none exists for posts.

## Task: post to a project, department, or person's feed

1. Resolve the numeric ID first, in its home category:
   `bitrix24_list_workgroups(query="Website Redesign")` (projects) → ID 5.
2. **✍** `bitrix24_create_post(message="Sprint review moved to Friday",
   title="Schedule change", dest="SG5")` → returned `id` confirms creation.

## Task: catch up on the feed / summarize someone's posts

1. Post rows hold only a numeric-string `AUTHOR_ID`, no name — resolve first:
   `bitrix24_search_users(query="Marina")` (organization) → user ID 1269.
2. `bitrix24_list_posts()` → filter client-side on `AUTHOR_ID == "1269"`.
   Every scalar is a string (`"ID": "217"`) — never compare against integers.
3. Post body is `DETAIL_TEXT`, not `MESSAGE`. Ordering is unverified — check
   `DATE_PUBLISH` on every page; never early-stop a date-bounded scan.
4. Feed rows include auto-published task and calendar entries — exclude or
   caveat them when reporting what a person "posted".

## Pagination

Fixed 50 per page — pass `start` in multiples of 50, or feed `next_cursor`
back as `start` until null; rate limit ~2 requests/second portal-wide.

## Reference — `dest` tokens

| Token | Audience | Resolve the numeric ID via |
|---|---|---|
| `UA` | All employees | — |
| `U<id>`, e.g. `U17` | One user | `bitrix24_list_users` / `bitrix24_search_users` (organization) |
| `SG<id>`, e.g. `SG5` | Workgroup/project | `bitrix24_list_workgroups` (projects) |
| `DR<id>`, e.g. `DR3` | Department | `bitrix24_list_departments` (organization; no name filter — match client-side) |
