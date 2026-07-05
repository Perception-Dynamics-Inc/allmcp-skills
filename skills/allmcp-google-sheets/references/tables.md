# Google Sheets tables — workflows

Native structured tables (typed columns, dropdown validation, banded rows). Only
add/update/delete exist. Steps marked **✍** change data (including the
cross-category pointers below) — confirm before firing if the request is vague.

## Model note — read first

- **No read/list tool here.** To get a `tableId`, a table's `range`, or the
  integer `sheetId` the `range` needs, FIRST call
  `google_sheets_get_spreadsheet(spreadsheet_id, include="full")` (`metadata`
  category) and read `sheets[].sheetId` and `sheets[].tables[]`. `include="full"`
  is mandatory — `summary` strips `tables[]`. Resolve IDs this way; never ask the
  user for a numeric `sheetId`/`tableId` or invent one.
- **`add_table` defines structure, not content.** Write data first via the
  `values` category (`USER_ENTERED`, real numbers not `"$1,200"`), then table the
  populated range. An empty range yields placeholder `Column 1, Column 2…`.
- **`range` is a GridRange** `{sheetId, startRowIndex, endRowIndex,
  startColumnIndex, endColumnIndex}` (standard half-open 0-based bounds);
  `sheetId` is the integer tab id, not the tab name. **`columnIndex` in
  `columnProperties` is table-relative** — the table's first column is `0` even
  if the table starts at sheet column C.

## Task: turn a populated range into a table

1. `google_sheets_get_spreadsheet(spreadsheet_id="1AbC…", include="full")` → tab `sheetId`.
2. **✍** `google_sheets_add_table(spreadsheet_id="1AbC…", table={"name":"Q2 Expenses",
   "range":{"sheetId":0,"startRowIndex":0,"endRowIndex":40,"startColumnIndex":0,"endColumnIndex":5},
   "columnProperties":[{"columnIndex":0,"columnName":"Date","columnType":"DATE"},
   {"columnIndex":2,"columnName":"Amount","columnType":"CURRENCY"}, …]})` → `tableId`.

Pick `columnType` by intent (table below); money is `CURRENCY`, not `DOUBLE`.
`name` must be unique across the spreadsheet; tables can't overlap.

A `DROPDOWN` column REQUIRES `dataValidationRule`, shape *exactly*:
`{"condition":{"type":"ONE_OF_LIST","values":[{"userEnteredValue":"Paid"},{"userEnteredValue":"Pending"}]}}`.
Do NOT add `strict`/`showCustomUi`/`inputMessage` (cell-level keys, invalid
here); non-DROPDOWN columns omit `dataValidationRule`. The `table` dict isn't
validated client-side — wrong keys surface only as a Google 400.

## Task: rename or extend a table (lossy — verify)

1. `google_sheets_get_spreadsheet(…, include="full")` → `tableId` + current `range`.
2. **✍** `google_sheets_update_table(spreadsheet_id="1AbC…", table={"tableId":"<step 1>",
   "name":"Q3 Pipeline","range":{…"endRowIndex":120…}}, fields="name,range")`. Set
   `fields` to only the keys you send; `"*"` clears `columnProperties` you omit.
3. Re-read with `google_sheets_get_spreadsheet(…, include="full")` and confirm the
   `range`. `update_table`/`delete_table` return a hard-coded `success=True`
   without reading back, so HTTP 200 never proves a range/footer edit landed.

## Task: remove the table but keep the data

**✍** `google_sheets_delete_table` already keeps the cell values. Do NOT chain a
`values` clear/delete unless the user explicitly wants the data gone too.

## Polish pipeline (other categories)

Cross-category pointers — all **✍** (they live in sibling categories, not here).
`columnProperties`/`rowsProperties` already give typed columns + banding/header
colors. Beyond that: freeze the header via `sheets`
(`google_sheets_update_sheet_properties`, `gridProperties.frozenRowCount`) →
auto-fit via `dimensions` (`google_sheets_auto_resize_dimension`) → color scales
via `conditional` → chart via `charts`. Banding on a non-table range has no tool
— use `google_sheets_batch_update` (`raw`) with `addBanding`.

## Reference — ColumnType

Several values fit the same data, so a wrong-but-valid pick degrades silently.

| Data | `columnType` |
|---|---|
| Money / dollars | `CURRENCY` (not `DOUBLE`) |
| Plain number | `DOUBLE` |
| Percentage | `PERCENT` |
| Date / time / datetime | `DATE` / `TIME` / `DATE_TIME` |
| Free text | `TEXT` |
| Checkbox | `BOOLEAN` |
| Fixed-choice list | `DROPDOWN` (needs the `ONE_OF_LIST` rule above) |

Also valid: `FILES_CHIP`, `PEOPLE_CHIP`, `FINANCE_CHIP`, `PLACE_CHIP`, `RATINGS_CHIP`.
