# Google Sheets named ranges — workflows

A named range is a stable label (e.g. `Q1_Sales`) bound to a cell block, usable in
formulas (`=SUM(Q1_Sales)`). This category only **writes**: add, update, delete.
Steps marked **✍** change the spreadsheet — confirm a vague request first.

## Model note — read before acting

- **This category has no read/list/get tool.** To see existing named ranges, or to
  get the `named_range_id` that `update`/`delete` require, call
  `google_sheets_get_spreadsheet(spreadsheet_id, include="full")` (**metadata**
  category) and read `namedRanges[]` — each entry has `namedRangeId`, `name`, and a
  `range` GridRange carrying the tab's integer `sheetId`.
- **`include="full"` is mandatory for that read. The default `include="summary"`
  strips `namedRanges[]` entirely** — you would see zero and wrongly report none
  exist. Never read named ranges in summary mode.
- **`update`/`delete` take only `named_range_id` (opaque string), never the human
  name** — look it up first.
- **Named ranges are spreadsheet-scoped, not tab-scoped.** The tab is implied only
  by the `sheetId` in the range. Don't promise per-tab name scoping.

## Task: create a named range (so formulas read =SUM(Q1_Sales))

1. **✍** `google_sheets_add_named_range(spreadsheet_id="1a2B…", name="Q1_Sales",
   sheet_name="Sheet1", start_row_index=0, end_row_index=20, start_column_index=0,
   end_column_index=1)` → returns the new `namedRangeId` as `id`. Pass `sheet_name`
   (auto-resolved) OR `sheet_id`, never neither.

Trust the returned `id` as confirmation — don't re-read in summary mode.

## Task: rename, repoint, or delete an existing named range

1. `google_sheets_get_spreadsheet(spreadsheet_id="1a2B…", include="full")` → scan
   `namedRanges[]` for the entry whose `name` matches → capture two distinct
   values from it: `namedRangeId` (opaque string) and, if repointing,
   `range.sheetId` (the tab's integer id). These are different fields — don't swap
   them.
2. Rename/repoint — **✍** `google_sheets_update_named_range(spreadsheet_id="1a2B…",
   named_range_id=<namedRangeId from step 1>, new_name="Q1_Revenue",
   new_range={"sheetId": <range.sheetId from step 1>, "startRowIndex": 1,
   "endRowIndex": 40, "startColumnIndex": 0, "endColumnIndex": 6})`. Pass
   `new_name`, `new_range`, or both — whichever you omit stays unchanged.
   `new_range` is a raw GridRange dict with no `sheet_name` convenience; embed the
   integer `sheetId` yourself.
3. Delete — **✍** `google_sheets_delete_named_range(spreadsheet_id="1a2B…",
   named_range_id=<namedRangeId from step 1>)`.

## Task: list what named ranges this sheet has

`google_sheets_get_spreadsheet(spreadsheet_id="1a2B…", include="full")` → present
each `namedRanges[]` entry's `name` and `range`. Summary mode returns none.

## Gotchas

- **Names that pass the local check but Google rejects (4xx):** cell-reference
  syntax (`A1`, `B20`, `R1C1`) and the booleans `true`/`false`. Substitute (e.g.
  `Block_A1`) if the user wants such a label.
- **Indices are 0-based half-open** (`start` inclusive, `end` exclusive): single
  cell A1 = `start_row_index=0, end_row_index=1, ...`, not `end=0`.
- **Duplicate `name` rejection is not guaranteed** — undocumented. Don't promise
  `add` will error on a collision; if uniqueness matters, list existing names with
  `include="full"` first.

## Cross-category links

- **Read the values inside a named range** → `google_sheets_values_get` (**values**
  category); the range name is a valid A1 reference there.
- **Protect a named range** → `google_sheets_add_protected_range` (**protection**
  category) accepts a `namedRangeId`.
