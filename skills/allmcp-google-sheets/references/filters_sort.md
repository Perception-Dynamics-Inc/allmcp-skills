# Google Sheets filters_sort — workflows

This category adds the filter bar, saves named filter views, and reorders rows — all via `batchUpdate`. Jump to the matching task; the bottom table holds the enum you can't guess. Steps marked **✍** change the sheet. Ranges are half-open 0-based GridRanges (`start` inclusive, `end` exclusive).

## Model note — read before acting

- **This category has NO read tool. Resolve every ID before you use it — never guess.** A filter/sort takes integers (column index, `sheetId`, `filterViewId`) that the user names by label ("the Status column", "the Sales tab", "the Pipeline view"). Guessing fails two ways: wrong column or wrong `sheetId` succeeds silently and filters the wrong data; a wrong `filterViewId` hard-errors. Resolve them FIRST, from the **metadata** and **values** categories:
  - **`sheetId` for a named tab and an existing view's `filterViewId`/`title`/`range`/`criteria`** → `google_sheets_get_spreadsheet(spreadsheet_id, include="full")` (**metadata**). Read `sheets[].properties.sheetId` and `sheets[].filterViews[]`. `include="summary"` strips `filterViews` — you must pass `"full"`. These are pure metadata; `include="full"` returns them WITHOUT cell values.
  - **The 0-based column index of a named column** ("the Status column") → you need the actual header cells, which `include="full"` does NOT contain. Read them: `google_sheets_values_get(spreadsheet_id, range="Sales!1:1")` (**values**) returns the header row as a list; the named column's position in it IS its index (these indices are SHEET-absolute). (Same data is reachable via `google_sheets_get_spreadsheet(..., include_grid_data=True)`, but `values_get` on one row is far leaner.)
- **`add_filter_view` has NO `sheet_name` param.** It takes only the `filter_view` dict, whose `range` is a GridRange requiring an integer `sheetId`. You must resolve the `sheetId` from metadata first — do not assume the tab is `sheetId=0` (gid 0 is only the first sheet ever created, often reordered or deleted). The other tools (`set_basic_filter`, `clear_basic_filter`, `sort_range`) DO accept `sheet_name`; prefer it there.
- **`filterViewId` is an integer everywhere** — returned as an int from add/duplicate, the int param on duplicate/delete, and the int value of `filter_view["filterViewId"]` on update. Never quote it. Thread the returned `id` straight through as a number.
- **Filter views vs basic filter vs sort.** A *filter view* is a saved, named, per-user view that does NOT change what teammates see — the collaborative-safe default for "share a filtered look". The *basic filter* is the single per-sheet filter bar everyone shares (one per tab; `set_basic_filter` silently overwrites the existing one). `sort_range` permanently reorders the real cells.
- **Capability boundaries — these live in OTHER categories:** freeze the header row + tab color → **sheets**; alternating-row banding / borders / number formats → **formatting** (banding needs the **raw** `google_sheets_batch_update` escape hatch); interactive on-canvas filter dropdowns (slicers) → **raw** only. Filtering hides rows, it does not delete them — to physically remove rows use **dimensions** or **find_replace**. Don't promise any of these from here.

## Task: add a sortable filter bar to a table

Adds the shared filter UI over the header+data range. The filter only reorders rows if you pass `sort_spec`.

1. `google_sheets_values_get(spreadsheet_id, range="Sales!1:1")` (**values**) → the header, giving the column count (for `end_column_index`) and the index of the column to sort by. `set_basic_filter` takes `sheet_name`, so you don't need the `sheetId`.
2. **✍** `google_sheets_set_basic_filter(spreadsheet_id, sheet_name="Sales", start_row_index=0, end_row_index=200, start_column_index=0, end_column_index=6, sort_spec={"dimensionIndex": 3, "sortOrder": "DESCENDING"})` — sorts by sheet column D (index 3), biggest first.

## Task: sort a table by a column (one-time, destructive)

`sort_range` physically reorders cells for everyone with **no undo** — the prior order is lost. On a vague "sort this", confirm before firing; for a reversible view prefer `set_basic_filter` or a filter view instead.

1. `google_sheets_values_get(spreadsheet_id, range="Orders!1:1")` (**values**) → resolve the sort column's sheet-absolute index from the header row (its position in the returned list).
2. **✍** `google_sheets_sort_range(spreadsheet_id, sheet_name="Orders", start_row_index=1, end_row_index=500, start_column_index=0, end_column_index=5, sort_column_index=4, order="DESCENDING")` — start at row index 1 to keep the header out of the sort.
3. Multi-key (e.g. region A→Z, then revenue high→low): add `extra_sort_specs=[{"dimensionIndex": 6, "sortOrder": "DESCENDING"}]`.

## Task: save a named filter view to share a filtered look

Filter views don't disturb teammates — the safe choice for "send a link showing only Acme's rows".

1. `google_sheets_get_spreadsheet(spreadsheet_id, include="full")` (**metadata**) → the tab's `sheetId` (`add_filter_view` has no `sheet_name`, so you need the integer). Then `google_sheets_values_get(spreadsheet_id, range="Sales!1:1")` (**values**) → the index of each column you'll filter on.
2. **✍** `google_sheets_add_filter_view(spreadsheet_id, filter_view={"title": "Acme — open", "range": {"sheetId": 812345678, "startRowIndex": 0, "endRowIndex": 200, "startColumnIndex": 0, "endColumnIndex": 6}, "criteria": {"2": {"condition": {"type": "TEXT_EQ", "values": [{"userEnteredValue": "Acme"}]}}, "4": {"hiddenValues": ["Closed"]}}})` → returns the new `filterViewId` (an int). Omit `filterViewId` — the server assigns it.

`criteria` is keyed by column index as a STRING (`"2"`, not `2`); `hiddenValues` and a `condition` can coexist on the same column and both must pass.

## Task: clone a saved view and tweak it

"Make a copy of the Q1 view for Q2" — resolve the source id from metadata, never hard-code it.

1. `google_sheets_get_spreadsheet(spreadsheet_id, include="full")` (**metadata**) → find the view named "Q1" in `sheets[].filterViews[]` → its `filterViewId`.
2. **✍** `google_sheets_duplicate_filter_view(spreadsheet_id, filter_view_id=<from step 1, int>)` → returns the new view's `id` (an int).
3. **✍** `google_sheets_update_filter_view(spreadsheet_id, filter_view={"filterViewId": <new id from step 2, int — unquoted>, "title": "Q2"}, fields="title")` — pass a NARROW `fields` mask. `fields` defaults to `"*"`, which replaces the whole view and clears the duplicated criteria/range you wanted to keep.

## Task: reset / remove filtering

- Shared filter bar: **✍** `google_sheets_clear_basic_filter(spreadsheet_id, sheet_name="Sales")`.
- A saved view: resolve its `filterViewId` from metadata (step 1 above), then **✍** `google_sheets_delete_filter_view(spreadsheet_id, filter_view_id=<int>)`.

## Gotchas

- **`add_filter_view`/`duplicate_filter_view` fail fast** if Google's reply lacks a `filterViewId` (raise rather than return a placeholder), so you never get an id that 404s later.
- **404 on a valid token** means the connected Google account can't see this spreadsheet → ask the user to share it with that account; never advise making the file public.

## Reference — enums

**`order` / `sortOrder`**: `ASCENDING` (A→Z, low→high; default) · `DESCENDING` (Z→A, high→low).

**`criteria[col].condition.type`** (filter-by-rule; each value is `{"userEnteredValue": "100"}`, or `{"relativeDate": "PAST_WEEK"}` for date types): text — `TEXT_EQ` `TEXT_NOT_EQ` `TEXT_CONTAINS` `TEXT_NOT_CONTAINS` `TEXT_STARTS_WITH` `TEXT_ENDS_WITH`; number — `NUMBER_GREATER` `NUMBER_GREATER_THAN_EQ` `NUMBER_LESS` `NUMBER_LESS_THAN_EQ` `NUMBER_EQ` `NUMBER_BETWEEN` (2 values) `NUMBER_NOT_BETWEEN`; date — `DATE_BEFORE` `DATE_AFTER` `DATE_ON_OR_BEFORE` `DATE_ON_OR_AFTER` `DATE_BETWEEN`; membership — `ONE_OF_LIST` (many values) `ONE_OF_RANGE`; presence — `BLANK` `NOT_BLANK` (take zero values); `CUSTOM_FORMULA` (1 value). `relativeDate`: `PAST_YEAR` `PAST_MONTH` `PAST_WEEK` `YESTERDAY` `TODAY` `TOMORROW`.
