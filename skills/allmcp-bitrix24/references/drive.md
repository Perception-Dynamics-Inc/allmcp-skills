# Bitrix24 Drive — workflows

Read-only category. Jump to the matching task; the bottom table holds the unguessable enums.

## Model note — read before acting

- **Tool names undersell the model.** `bitrix24_list_folders` lists *storages*
  (whole drives: Company Drive, personal, workgroup), not folders;
  `bitrix24_list_files` lists a folder's *children — files AND subfolders
  mixed*, split on `TYPE` (`"folder"`/`"file"`).
- **Storage ID ≠ folder ID.** They are separate ID spaces. Enter a drive via
  its `ROOT_OBJECT_ID` — never its `ID` — as `folder_id`; a storage `ID` may
  hit the wrong folder or error (untested — don't find out).
- **Metadata only, no search.** Hand over a file's `DOWNLOAD_URL`, but you
  cannot read file contents, mint public/share links, or filter by name —
  finding a file means walking the tree (recipe below). Say so plainly
  instead of improvising.

## Task: browse a drive or find a file and get its download link

1. `bitrix24_list_folders()` → select the storage by `ENTITY_TYPE` +
   `ENTITY_ID`, never by `NAME` (the list may hold a `user` storage per employee).
   Company Drive = the `"common"` storage. "My drive" = the `"user"` storage
   whose `ENTITY_ID` equals the `ID` from `bitrix24_get_current_user()`
   (`organization` category). A project's drive = the `"group"` storage whose
   `ENTITY_ID` equals the workgroup `ID` from
   `bitrix24_list_workgroups(query="Marketing")` (`projects` category).
2. `bitrix24_list_files(folder_id=<step 1 storage's ROOT_OBJECT_ID>)` → scan
   `NAME`; descend by passing each `TYPE: "folder"` row's `ID` as the next
   `folder_id`. A walk costs one call per folder per 50-row page against a
   ~2 req/s portal-wide limit — page sequentially, never in parallel; ask the
   user for the likely folder before a deep walk; expect
   `QUERY_LIMIT_EXCEEDED`/503 if you burst.
3. `bitrix24_get_file(file_id=<ID of the TYPE: "file" row>)` → `DOWNLOAD_URL` —
   token-bearing; hand it to the requesting user only, never as a public/share link.
4. Nothing found? Listings omit anything the webhook user lacks Read access
   to — report "not found *or not visible to this connection*".

## Task: report who created or last edited a file

`CREATED_BY`/`UPDATED_BY` on any file are numeric user IDs — resolve each via
`bitrix24_get_user(b24_user_id=<that ID>)` (`organization` category); never
answer "user #42". (`bitrix24_search_users` is fulltext over names/positions —
it cannot look up a numeric ID.) The edit timestamp is `UPDATE_TIME` (ISO 8601).

## Gotchas

- `include` is a no-op here — `"summary"` and `"full"` are identical and
  `DOWNLOAD_URL` always survives; never retry with `efficiency_on=False` to "recover" a field.
- Rows with `DELETED_TYPE` other than `"0"` are in the trash — skip or flag
  them, don't report them as live files.

## Reference — enums (system-defined)

| Field | Values |
|---|---|
| storage `ENTITY_TYPE` | `user` = personal drive (`ENTITY_ID` = user ID) · `common` = Company Drive · `group` = workgroup drive (`ENTITY_ID` = workgroup ID) |
| child `TYPE` | `folder` → `ID` feeds `bitrix24_list_files` · `file` → `ID` feeds `bitrix24_get_file` |
| `DELETED_TYPE` | `0` active · `3` trashed · `4` deleted with parent folder |
