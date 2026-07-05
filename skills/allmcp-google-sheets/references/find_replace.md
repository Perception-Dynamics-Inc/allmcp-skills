# Google Sheets find_replace — workflows

Data-cleanup tools, not design tools. Jump to the matching task. Steps marked
**✍** mutate the sheet; all four are immediate with no dry-run or undo, so
confirm scope first. They return only a count in `message`, never cells —
verify any lossy write by reading the range back with `google_sheets_values_get`
(values category), the only way to confirm what actually changed.

## Model note — read before acting

- **Edits text, not type or format.** find_replace rewrites cell *strings*; it
  cannot change number format or data type. Stripping `$`/`,` from a Plain-text
  column leaves text that still won't `=SUM` — forcing numeric type is the
  **formatting** category's job. Promise the text edit, not a working number.
- **`scope` defaults to `all` — every tab.** "Replace X with Y" with no tab named
  rewrites the WHOLE workbook. Confine it with `scope='sheet'` (one tab) or
  `scope='range'` (one rectangle) plus the tab.
- **Tabs are an integer `sheetId`, not a name.** Pass `sheet_id`, or `sheet_name`
  for an **exact, case-sensitive** match — `"deals"`, `"Deals "`, or wrong case
  is a hard error, not a no-op. If the title is uncertain, read the `sheetId`
  from `google_sheets_get_spreadsheet` (metadata category, `include='summary'`
  returns `sheets:[{sheetId,title,rowCount,columnCount}]`) and pass that — the
  same call gives `columnCount`/`rowCount` to size a dedupe/randomize range to
  the real table instead of guessing.
- **Indices are 0-based, half-open** (`start_row_index=1` skips the header). No
  find-only/highlight mode exists — to flag duplicates instead of deleting, use
  the **conditional** category.

## Task: fix a typo / rename a value across the whole workbook

1. **✍** `google_sheets_find_replace(spreadsheet_id="<id>", find="Calfornia",
   replacement="California", match_entire_cell=True)` — `scope` defaults to
   `all`. Use `match_entire_cell=True` for whole-value swaps; the substring
   default also rewrites "Calfornian", and "1"→"2" turns "2021" into "2022".
2. A `message` of `"Replaced 0 occurrence(s)."` is SUCCESS, not an error — a knob
   is off: toggle `match_case`, or set `include_formulas=True` to also search
   formula text (off by default, it matches displayed values only). Confirm the
   intended cells changed with a `google_sheets_values_get` read-back.

## Task: standardize one column on one tab only

**✍** `google_sheets_find_replace(spreadsheet_id="<id>", find="N/A",
replacement="", scope="range", sheet_name="Leads", start_row_index=1,
end_row_index=5000, start_column_index=4, end_column_index=5)` — column E only
(index 4, half-open), below the header. Empty `replacement` blanks the match.
Resolve the `sheetId` first if "Leads" isn't the exact title.

## Task: strip a currency symbol / units so numbers can compute

1. **✍** `google_sheets_find_replace(spreadsheet_id="<id>", find="\\$",
   replacement="", search_by_regex=True, scope="range", sheet_name="Sales",
   start_row_index=1, end_row_index=1000, start_column_index=2,
   end_column_index=3)` — regex is the canonical strip; `find` is **Java regex**
   and `replacement` group refs are `$1`, `$2` (not `\1`). Confine to the price
   column so symbols elsewhere survive.
2. Read back with `google_sheets_values_get`: still-left-aligned cells are
   Plain-text-formatted — find_replace can't fix that; hand the number-format
   step to the **formatting** category.

## Task: clean a messy import, then de-dupe on a key column

Trim FIRST — `delete_duplicates` treats `"a@x.com "` and `"a@x.com"` as different
rows, so untrimmed near-duplicates survive.

1. **✍** `google_sheets_trim_whitespace(spreadsheet_id="<id>",
   sheet_name="Contacts", start_row_index=1, end_row_index=5000,
   start_column_index=0, end_column_index=6)` — strips leading/trailing
   whitespace; it may also collapse interior runs to one space (the menu does,
   our tool is not bench-verified on it). Do not rely on interior collapse to
   merge `"Acme  Co"` and `"Acme Co"` — confirm with the step-3 read-back.
2. **✍** `google_sheets_delete_duplicates(..., start_column_index=0,
   end_column_index=<columnCount>, compare_column_indices=[2])` — compares only
   the email column (absolute index 2, required, min 1). For whole-row
   duplicates pass every column index. The RANGE must span the full used table
   width: deleting a row shifts only cells inside the range, so a range narrower
   than the table leaves orphaned cells in the columns beyond it. Do not
   hardcode the width — read `columnCount` from `google_sheets_get_spreadsheet`
   (metadata) and use it as `end_column_index`.
3. Read back with `google_sheets_values_get` to confirm survivors.

Surface before step 2: **case still splits rows** (`"A@x.com"` ≠ `"a@x.com"`;
find_replace cannot lowercase a column — true case-folding needs the values or
**raw** category), and **dedupe keeps the TOPMOST row, never the newest** — for a
CRM master where the freshest record should win, sort descending first with
`google_sheets_sort_range` (filters_sort category).

## Task: shuffle rows for a sample, raffle, or fair ordering

**✍** `google_sheets_randomize_range(spreadsheet_id="<id>", sheet_name="Entrants",
start_row_index=1, end_row_index=500, start_column_index=0,
end_column_index=<columnCount>)` — span the full table width (from
`google_sheets_get_spreadsheet` `columnCount`); a too-narrow range shuffles only
those columns and desyncs the rest of each row. Returns only `"Range
randomized."`: no count, no seed, no undo. The only proof it ran is to read the
range back with `google_sheets_values_get`.

## Out of scope — route elsewhere

No styling here (number formats, borders, banding, color scales, charts). For
"clean it up and make it presentable", finish the cleanup, then hand the design
half to **formatting** (number formats, styled header), **conditional**
(value-driven color), and **charts**. Anything else (textToColumns, pasteData,
fuzzy or keep-last dedupe) needs `google_sheets_batch_update` (raw category).
