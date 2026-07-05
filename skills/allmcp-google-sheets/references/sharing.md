# Google Sheets sharing — workflows

Sharing is Drive's whole-file access control on a spreadsheet's `file_id`. Jump
to the matching task. **✍** marks steps that change who can access the file —
on a vague request, confirm the person, role, and file first.

## Model note — read before acting

- **You need a `file_id`, not a name.** Resolve "Q3 Budget" via
  `google_sheets_list_spreadsheets` (the `files` category) before any call here.
- **Map the user's words to a `role`:** Viewer → `reader`, Commenter →
  `commenter`, Editor → `writer`. `role='owner'` is an ownership transfer and
  needs `transfer_ownership=True`.
- **A `permission_id` is opaque and comes only from `list_permissions`** — never
  derivable from an email. To change or revoke access, list first, match the
  person by `emailAddress`/`displayName`, then use that `id`.
- **`update_permission` patches the role only.** `type` and grantee are immutable
  — to re-target a grant (different person, or user→anyone), `remove_permission`
  it and `share_spreadsheet` anew.
- **Consumer `@gmail.com` ownership transfer is impossible here** — it needs a
  two-step pending-owner consent flow these tools can't drive. Don't promise or
  attempt it; offer `writer` (full edit) instead. Only `owner` transfer to an
  address in the user's *own Workspace org* works one-shot.

## Task: share with a person (view / comment / edit)

1. `google_sheets_list_spreadsheets(name_contains="Q3 Budget")` (`files`) → `file_id`.
2. **✍** `google_sheets_share_spreadsheet(file_id=<step 1>, type='user',
   role='writer', email_address='marina@acme.com',
   email_message="Please review by Fri")` → permission `id`. Default
   `send_notification_email=True` emails her; the note rides that email.
3. `google_sheets_get_permission(file_id, permission_id=<step 2 id>)` → confirm
   `role` and `emailAddress` stuck.

Use `type='group'` (group's email) for a Google Group, or `type='domain'` +
`domain='acme.com'` for a whole Workspace org.

## Task: make a view-only public link

1. **✍** `google_sheets_share_spreadsheet(file_id=<resolved>, type='anyone',
   role='reader')` — omit `email_address`/`domain`. No email is sent for
   `anyone`; the listed grant carries no `displayName`.
2. The tool returns a `permission_id`, not a URL. Get the link from
   `google_sheets_get_file_metadata(file_id)` (`files`) → `webViewLink`. Do not
   hand-build a `docs.google.com/...` URL. For "company only, not the whole
   internet," prefer `type='domain'`.

## Task: audit, change, or revoke access

1. `google_sheets_list_permissions(file_id=<resolved>)` → match the grant by
   `emailAddress`/`displayName`; a public link shows as `type='anyone'`. There is
   no `total` — page until `next_cursor` is null before concluding "no access".
2a. Change a level: **✍** `google_sheets_update_permission(file_id,
   permission_id=<step 1>, role='writer')`. Never demote the connected user's own
   grant — it can lock them out.
2b. Revoke: **✍** `google_sheets_remove_permission(file_id,
   permission_id=<step 1>)` → `success=True` (Drive 204, no body).
3. Re-run `list_permissions` to confirm what stuck — no mutating tool re-reads.

## Task: transfer ownership

Only to an address in the user's **own Workspace org**:
`google_sheets_share_spreadsheet(file_id=<resolved>, type='user', role='owner',
email_address='colleague@acme.com', transfer_ownership=True)`. The old owner
drops to `writer`; the notice email can't be suppressed. For a personal
`@gmail.com` recipient, stop and offer `role='writer'` instead.

## Gotchas

- **404 = the connected account can't see the file**, not that it's gone — it
  belongs to a different Google account or the `file_id` is wrong. Fix by sharing
  it *to* the connected account or reconnecting. Never make it public.
- **Serialize writes on one file** — concurrent share/update/remove on the same
  `file_id` aren't supported; only the last wins.

## Reference — role enum

| User says | `role` |
|---|---|
| Viewer / view-only | `reader` |
| Commenter | `commenter` |
| Editor / can edit | `writer` |
| Owner (transfer) | `owner` (needs `transfer_ownership=True`) |

`organizer`/`fileOrganizer` apply only to shared-drive files (not targeted here).
`type`: `user`/`group` need `email_address`; `domain` needs `domain`; `anyone`
is a public link (omit both).
