# Google Sheets values — workflows

Jump to the task matching the user's request. This category reads and writes
cell *content* only. Steps marked **✍** change the user's spreadsheet — if the
request was vague, confirm specifics first. The Model note prevents the costliest
mistakes; read it before acting.

## Model note — read first

- **This category writes content, never appearance.** No colors, fonts, number
  formats (`$#,##0.00`, `0.0%`, `yyyy-mm-dd`), borders, merges, frozen rows,
  banding, column widths, or conditional colors are reachable from any
  `values_*` tool. "Make it look professional" is impossible here alone — style
  in the `formatting`, `dimensions`, `conditional`, `charts`, and `tables`
  categories (alternating-row banding only via `google_sheets_batch_update` in
  the `raw` category; no convenience tool exists). Never fake formatting by
  writing a display string.
- **Store real values, not pre-formatted strings.** Write `1200`, not
  `"$1,200.00"`; write `0.12`, not `"12%"`; write real dates under
  `USER_ENTERED`. Currency/percent strings break `=SUM` and every number format
  applied later — a number format like `$#,##0` (set in `formatting`) is what
  renders the real `1200` as `$1,200`.
- **A PUT update overwrites only the cells you send — it never blanks the
  tail.** `values_update` with fewer rows than the old block leaves the old
  trailing rows in place. See "Replace a region" — the #1 data-integrity trap.
- **`USER_ENTERED` (the default) is what you want** — formulas evaluate, numbers
  and dates parse. Use `RAW` only to keep a literal string starting with `=`,
  `+`, or a leading zero.
- **Quote tab names containing spaces, and match case exactly:**
  `'Q1 Sales'!A1:C10`, not `Q1 Sales!A1`; `Sheet1` ≠ `sheet1`. An unquoted
  spaced name or wrong case 400/404s before the write runs.
- **IDs are unguessable.** Find the spreadsheet ID in the `files` category (Drive
  list/search; the Sheets API has no list). An integer `sheetId` (only for a
  `gridRange` filter) comes from `google_sheets_get_spreadsheet` (`metadata`).

## Task: dump a dataset into a sheet

1. **✍** `google_sheets_values_update(spreadsheet_id="1AbC…", range="'Q1 Sales'!A1",
   values=[["Date","Item","Amount"],["2026-01-04","Widget",1200],["2026-01-05","Gadget",980]])`.
   Anchor at the top-left cell; the array's shape sets the extent.
2. To add a total, do NOT guess the last data row. Either **✍** append the total
   row (lands after the real last row), or `values_get` the column first to learn
   the extent, then write `=SUM(A2:A<lastrow>)`. The write response's
   `updatedRange` reports the exact extent written.

## Task: append a row to a running log / tracker

**✍** `google_sheets_values_append(spreadsheet_id="1AbC…", range="Leads!A:E",
values=[["Jane Doe","jane@acme.com","Website",0.25,"2026-02-01"]],
insert_data_option="INSERT_ROWS")`. Pass a **tight** range bounding exactly the
table: append scans it for the last row and writes after it, so a loose range
over blank rows can misplace the row. The result is only `success` + a message
with the written range — it does NOT surface where the table was detected, so if
placement matters, `values_get` the area afterward to confirm the row landed
where you expect. Use `INSERT_ROWS` whenever anything lives below the table.
append has no positional insert — for a fixed slot, `values_update` an explicit
range instead.

## Task: read a range to summarize or compute

`google_sheets_values_get(spreadsheet_id="1AbC…", range="Budget!B2:B50",
value_render_option="UNFORMATTED_VALUE")` → real numbers (`1234`, `0.125`), not
display strings (`"$1,234.00"`) that break numeric parsing. Add
`date_time_render_option="FORMATTED_STRING"` for human dates (else dates read as
serial numbers); use `value_render_option="FORMULA"` to read formula text. For
several ranges use `google_sheets_values_batch_get` (and
`google_sheets_values_batch_update` for writes) — one quota unit each, far faster
than N single calls.

## Task: replace a region wholesale

A shorter `values_update` does NOT clear the old block's tail. To replace
`Pricing!B2:E50` (50 rows) with only 30 new rows:

1. **✍** `google_sheets_values_clear(spreadsheet_id="1AbC…", range="Pricing!B2:E50")`
   — clear the FULL OLD extent first.
2. **✍** `google_sheets_values_update(…, range="Pricing!B2", values=<30 rows>)`.

Alternative: pad the full old range with explicit `""`. `values_clear` keeps
formatting and data validation — to wipe those too, use `clear_formatting` in
the `formatting` category.

## Verify after every write

Writes have no per-range success map — `success=True` and cell counts prove
nothing about *what* landed, and a multi-range batch may partially apply on
error. After any write that matters, re-read with `values_get` /
`values_batch_get` using **`value_render_option="UNFORMATTED_VALUE"`** and
confirm the cells hold what you sent. The default `FORMATTED_VALUE` echoes
display strings and hides a string-vs-number defect (a faked `"12%"` reads back
as `"12%"`, looking fine).

## Gotchas

- **`""` clears a cell; `null`/`None` skips it (leaves it unchanged).**
  `[["A", None, "C"]]` writes A and C, leaves the middle as-is; `[["A","","C"]]`
  blanks the middle. Top write surprise.
- **values reads do not paginate** — bound every read by its A1 range; don't
  invent a `pageToken`. Quotas ~60 read + 60 write/min per user; on 429
  (`RESOURCE_EXHAUSTED`) the connector surfaces the error, so back off and batch.
- **A 404 with a valid connection** usually means the connected Google account
  can't see the file (wrong account or ID) — not a public-sharing issue. Share
  the file with the connected account; do not make it public.
- **In a DataFilter** (the `*_by_data_filter` tools) set exactly ONE key:
  `a1Range` (sheet NAME), `gridRange` (integer `sheetId`), or
  `developerMetadataLookup` — mixing them is a 400.
