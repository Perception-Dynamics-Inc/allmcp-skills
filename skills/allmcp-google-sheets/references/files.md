# Google Sheets files — workflows

This category is the Drive-side file layer: list/search, file metadata, create,
copy, rename, trash/untrash, permanent delete. It is where almost every task
starts — you resolve a **spreadsheet ID here**, then act on its contents in other
categories. Jump to the matching task. Steps marked **✍** create, move, or delete
the user's files — if the request was vague, confirm before firing one.

## Model note — read first

- **An empty `list_spreadsheets` result is ambiguous, not proof of absence.**
  With a narrow OAuth scope Drive returns HTTP 200 with an empty list even when
  matching sheets exist. Before saying "no such file," treat the result as
  *nothing visible to this account* and, if the user is sure it exists, suggest
  reconnecting for broader Drive read access — don't declare it gone.
- **This category sees only spreadsheets, and cannot touch folders or sharing.**
  No tool creates a folder, moves a file between folders, looks a folder up, or
  changes who has access. The only in-category way to place a file in a folder is
  `copy_spreadsheet(parent_folder_id=...)`, and **that folder ID must already be
  known** (from the user or a Drive URL). Do not reach for a second Drive
  integration to find folders. Permissions live in the `sharing` category;
  tab-level rename/duplicate in the `sheets` category.
- **There is no whoami tool.** Each file carries `ownerEmail`, but nothing returns
  the connected account's own address, so you cannot compute "the one I made." For
  "my budget sheet," disambiguate by name with the user or sort by
  `modifiedTime desc` and confirm — never assume the owner is the connected user,
  and don't silently take `items[0]`.
- **`get_file_metadata` is Drive metadata, not content** — owner, timestamps,
  parents, size. For tab names/`sheetId`s use `google_sheets_get_spreadsheet`
  (`metadata`); for cells use the `values` category.
- **Return shapes** (don't guess fields): list/get rows =
  `{id, name, mimeType, *Time, ownerEmail, webViewLink, parents}` (share link =
  `webViewLink`); create/copy = `{id, message}`; trash/untrash/rename/delete =
  `{success, message}`.

## Task: find a spreadsheet by name (resolve its ID)

`google_sheets_list_spreadsheets(name_contains="budget")` → scan rows, pick the
one the user means → `id`. `name_contains` is a case-insensitive substring on the
file name only (no content search); omit it to list all, but `""`/whitespace is
rejected. Add `include_trashed=True` for trashed files; on an empty result apply
the scope caveat (Model note #1) before reporting the file missing.

## Task: build a professional spreadsheet from scratch (the polish pipeline)

A polished sheet is assembled across categories — `files` only creates the empty
shell, then the formatting/structure work lives elsewhere. Create the ID here,
then run the pipeline in order:

1. **✍** `google_sheets_create_spreadsheet(title="Q3 Tracker",
   sheet_titles=["Summary","Data"])` → new `spreadsheet_id`. The file lands in
   **My Drive root** (no folder param; there is no move tool).
2. `google_sheets_get_spreadsheet(spreadsheet_id=<from step 1>)` (`metadata`) →
   confirm the first tab's name and integer `sheetId` for later steps.
3. **✍** Write data via the `values` category (`USER_ENTERED`; store **real
   numbers** like `1200`, never `"$1,200.00"` — pre-formatted strings break
   `=SUM` and every later number format).
4. Polish, each in its own category: number formats + styled header + sparing
   borders → `formatting`; freeze header + tab color → `sheets`; auto-size columns
   → `dimensions`; value-driven cell color → `conditional`; a chart → `charts`.
   Alternating-row banding has **no convenience tool** — only raw
   `google_sheets_batch_update` (`addBanding`) in the `raw` category.

## Task: copy a template for a new client

One call clones, renames, and places into a folder:

**✍** `google_sheets_copy_spreadsheet(file_id=<template id>,
new_title="Acme — Q3 Onboarding", parent_folder_id=<known folder id>)` → new
`id`. The copy is fully independent — own ID, sharing, revision history — so
editing it never touches the template. Omit `parent_folder_id` to land in My
Drive root; the folder ID must already be known (this category cannot look it up).

## Task: delete a spreadsheet

A plain "delete" means recoverable trash — **✍**
`google_sheets_trash_spreadsheet(file_id="1AbC…")` (undo with
`google_sheets_untrash_spreadsheet`). Reach for
`google_sheets_delete_spreadsheet` only when the user explicitly asks for
*permanent* removal; it bypasses trash and is irrecoverable. (For a *file* rename
use `google_sheets_rename_spreadsheet`; a *tab* rename is
`google_sheets_rename_sheet` in the `sheets` category.)

## Pagination

Only `list_spreadsheets` paginates, and **`total` is always `None`** — Drive
reports no count, so you cannot bound a search by size. Pass the prior call's
`next_cursor` as `page_token` and repeat until `next_cursor` is null; a match can
sit on page 2. `page_size` is 1–1000 (default 100).

## Gotchas

- **404 with a valid connection means the connected Google account can't see the
  file — wrong account or wrong ID, not a sharing-visibility bug.** Have the user
  share the file with the connected account; **never advise making it public.**
- **`modifiedTime desc` (the sort default) is not "created" or "made last
  quarter."** A recently touched old file sorts to the top; don't read `items[0]`
  after a sort as "the file the user means."
