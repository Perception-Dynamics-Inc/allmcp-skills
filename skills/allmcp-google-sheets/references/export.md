# Google Sheets export — workflows

`google_sheets_export_spreadsheet` is the only tool: it downloads a whole
spreadsheet in one format. The category is **read-only** — it never changes the
source file, so no step needs confirmation.

## Model note — read before acting

- **`csv` and `tsv` export only the FIRST tab**, never the whole workbook (a
  Drive limitation, not a bug). When the user wants *every* tab in one file, pick
  `xlsx` or `ods` (full workbook); `pdf` for a print/share snapshot; `html_zip`
  for a zipped HTML render. Never promise "all tabs as CSV" — it is impossible in
  one call, and there is no per-tab or per-range export, only the whole file.
- **`file_id` is the Drive file ID — the same value the rest of the connector
  calls `spreadsheet_id`.** If you only have a name, resolve it first in the
  `files` category: `google_sheets_list_spreadsheets(name_contains=...)` → `id`.
  This category cannot look a file up by name.
- **The result is the file's bytes as base64, not a download link.** `item`
  carries `{file_id, mime_type, format, size_bytes, content_base64}`. To
  reconstruct the file, base64-decode `content_base64` with the runtime's
  decoder; there is no URL to hand the user.
- **There is a ~10 MB cap on the returned payload.** A larger export fails with a
  clear error — fall back to `csv` (first tab only) or split the data, since
  there is no range-limited export.

## Task: export a spreadsheet to a file

1. Resolve the `file_id` (Model note 2) — from the user, a Drive URL
   (`https://docs.google.com/spreadsheets/d/{ID}/edit`), or the `files` category.
2. `google_sheets_export_spreadsheet(file_id="1AbC…", format="xlsx")` → read
   `content_base64` from `item` and base64-decode it to get the file. Use
   `format="csv"` only when the user explicitly wants just the first tab as plain
   text.
