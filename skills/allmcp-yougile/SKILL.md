---
name: allmcp-yougile
description: Work YouGile — agile projects, Kanban boards, columns, tasks, chats, stickers — through the AllMCP hub. Use when the user asks to read or change anything in their YouGile company (projects, boards, tasks, deadlines, assignees, chat messages, sticker labels, sprints) via AllMCP. NOT for calling YouGile's REST API directly with your own code, or company administration (billing, inviting/removing employees, revoking keys).
---

# YouGile via AllMCP

YouGile is an agile project-management platform. Through AllMCP your agent
gets namespaced tools (`yougile_list_tasks`, `yougile_create_board`, …) — 36
of them, organized into six categories. This file is orientation and routing;
the real playbooks are in `references/` — read the matching one **before**
chaining tools in a category you haven't used this session.

Platform mechanics (endpoint, connecting, hidden tools, quota) live in the
`allmcp` skill — install it alongside this one.

## Connect

- Provider key: `yougile` (exactly this).
- Credential: a YouGile **API key**, passed as the single `api_key` field of
  `connect_provider`. The user creates it inside YouGile: open
  [app.yougile.com](https://app.yougile.com), log into the target company,
  press `Ctrl + ~` (or open the Configurator / admin gear icon) → **Keys**
  tab → **Create key**, and copy the JWT-style token (`eyJhbGciOi...`).
- The key is **company-scoped** and stays valid until revoked or the user
  leaves the company. It is a live credential — never echo it back in full or
  log it; if it leaks, tell the user to revoke it on the Keys tab.
- After connecting, `tasks` and `projects` tools are advertised right away;
  boards, columns, chat, and stickers unlock via
  `describe_category("yougile", ...)` or on first call.

## The three rules that prevent most failures

1. **Everything hangs off the ID chain: project → board → column → task.**
   Resolve down the chain with the `list_*` tools — a board needs a
   `project_id`, a column needs a `board_id`, and a task **requires** a
   `column_id` (there is no inbox or default column). Never pass a name where
   an ID is expected.
2. **Updates replace, they don't merge.** `users` on projects and group
   chats, `assigned` and `stickers` on tasks, chat subscribers, sprint
   `states` — all full replacements. Read the current value first
   (`include="full"` on the matching `get_*` tool), merge your change in,
   then write; otherwise you silently drop existing members/assignees.
3. **Deletion is soft everywhere.** `deleted=True` hides a record,
   `deleted=False` restores it, and hidden records only appear with
   `include_deleted=True`. Nothing in these tools destroys data permanently —
   say "archive", not "delete", when reporting to the user.

## Categories → playbooks

Read the reference file before working in a category; each one carries the
unguessable enums (role values, palette colors 1–16, ms-epoch deadlines), the
ID-flow between tools, and the boundaries.

| Category | Read | Covers |
|---|---|---|
| `projects` | [references/projects.md](references/projects.md) | Top-level workspaces, membership & roles — the root of the ID chain |
| `boards` | [references/boards.md](references/boards.md) | Kanban boards inside a project |
| `columns` | [references/columns.md](references/columns.md) | Board columns — the mandatory target for every task |
| `tasks` | [references/tasks.md](references/tasks.md) | Create/move/complete tasks, deadlines, assignees, sticker assignment, chat subscribers |
| `chat` | [references/chat.md](references/chat.md) | Group chats and task chat threads — a task ID doubles as its `chat_id` |
| `stickers` | [references/stickers.md](references/stickers.md) | String label stickers and sprint stickers, plus their states |

## Boundaries worth knowing up front

- **There is no user directory tool.** Resolve people's IDs from existing
  data — project membership (`yougile_get_project` with `include="full"`) or
  task assignees — you cannot look a user up by name or email.
- Every task has a built-in chat (its task ID is the `chat_id`), but task
  chats can't be renamed or deleted independently of the task.
- `yougile_list_tasks` has no project-wide filter — narrow by column or
  assignee, or sweep board by board.
- No file/attachment surface — these tools don't upload or download files.
