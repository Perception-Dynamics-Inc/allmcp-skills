# Google Sheets dimensions — workflows

This category changes row/column **structure and size**: insert, delete, resize, auto-fit, append, move. Jump to the matching task. Steps marked **✍** mutate the grid for everyone and **cannot be undone** — there is no revert tool, so confirm specifics on a vague request before firing one. The tool schemas already carry the enum (`ROWS`/`COLUMNS`), the 0-based half-open index math, and `inherit_from_before` legality — trust them; this doc is about getting the *target right* before you mutate.

## Model note — read before acting

- **These tools target by integer index only — never by header name or A1 letter.** "Widen the Revenue column", "move the Region column to the front", "delete the Notes column" all require you to resolve the header to a 0-based column index YOURSELF first. There is no read tool in this category and no fuzzy targeting — a guessed index is a **silent mis-mutation**: a wrong column to `resize`/`delete`/`move` has valid indices, so the API succeeds and corrupts the wrong column without erroring.
- **Resolve the target before every mutation:** call `google_sheets_get_spreadsheet` (**metadata**, `include="summary"`) for the tab's `sheetId` + `rowCount`/`columnCount`, and read row 1 via `google_sheets_values_get` (**values**) to map header text → column index. Column A=0, B=1, … Z=25, AA=26.
- **Destructive vs recoverable asymmetry.** Guessing an index on `resize`/`auto_resize` is recoverable (re-run with the right index, no data lost). Guessing on `delete_dimension` or `move_dimension` is permanent and unverifiable here — for those two, the resolved index must be **confirmed, never guessed**.
- **No file discovery here.** This category cannot find a spreadsheet by name/URL. Get the `spreadsheet_id` up front; an unknown id returns a **404** (the connected Google account can't see it — ask the user to share with that account, never advise making the file public). Don't fire a mutation with a placeholder id.
- **No read-back exists in this category.** After a layout-critical resize or auto-fit, confirm the result with a cross-category read (`google_sheets_get_spreadsheet` / `google_sheets_values_get`) — `success=True` only means the request was accepted, not that the layout looks right.

## Task: auto-fit columns so nothing is cut off

The most common cleanup (the API "double-click the border"). Fit the **used** range, not a guessed A–Z.

1. `google_sheets_get_spreadsheet(spreadsheet_id, include="summary")` (**metadata**) → the tab's `sheetId` and `columnCount`.
2. **✍** `google_sheets_auto_resize_dimension(spreadsheet_id, sheet_id=<step 1>, dimension="COLUMNS", start_index=0, end_index=<columnCount from step 1>)`.

Over-covering empty trailing columns is harmless (they collapse to default); **under**-covering silently leaves the right edge un-fitted, so always use the real `columnCount`, never a guess. For a print/screenshot tighten, run again with `dimension="ROWS"` — but note auto-fit on ROWS **resets then regrows** heights, discarding any hand-set row height.

## Task: give columns sensible widths / set the header row taller

Fixed pixel sizing for a clean report. `pixel_size` is bounded 1–10000.

1. Resolve which column indices are which header (Model note) before sizing a named column.
2. **✍** `google_sheets_resize_dimension(spreadsheet_id, sheet_id=<id>, dimension="COLUMNS", start_index=<resolved>, end_index=<resolved+1>, pixel_size=140)`.

Tasteful widths: narrow ID/index ~60–80 px, data columns ~120–160 px, a wide text/notes column ~250–320 px; header row height ~28–32 px. If you also auto-fit, auto-fit FIRST then set fixed widths — auto-fit would otherwise overwrite them. Then read back the layout via **metadata**/**values** if the result must be exact (auto-fit's interaction with wrapped/merged cells is undocumented).

## Task: insert blank rows/columns at a position

1. **✍** `google_sheets_insert_dimension(spreadsheet_id, sheet_id=<id>, dimension="ROWS", start_index=4, end_index=7, inherit_from_before=true)` — inserts 3 blank rows before row index 4, copying formatting from the row before.
2. `inherit_from_before=true` is rejected when inserting at index 0 (no "before" row/col to copy) — leave it `false` there.

These insert **blank structure only**. To put values in, follow with **values** (`google_sheets_values_update`). For appending a *data* row, prefer `google_sheets_values_append` with `INSERT_ROWS` (**values**) — it auto-grows the grid, so you usually don't need `append_dimension` at all.

## Task: delete or move rows/columns

Both are permanent and silent-on-wrong-target. Resolve and **confirm** the index before firing.

- **Delete:** confirm the resolved range, then **✍** `google_sheets_delete_dimension(spreadsheet_id, sheet_id=<id>, dimension="COLUMNS", start_index=<resolved>, end_index=<resolved+1>)`. (For removing duplicate *rows*, that's `delete_duplicates` in **find_replace**, not this.)
- **Move:** **✍** `google_sheets_move_dimension(spreadsheet_id, sheet_id=<id>, dimension="COLUMNS", start_index=2, end_index=3, destination_index=0)` — moves the column at index 2 to the front.

## Task: pad the sheet with blank rows at the bottom

**✍** `google_sheets_append_dimension(spreadsheet_id, sheet_id=<id>, dimension="ROWS", length=500)` — always appends at the END (no position, no indices). To append rows *with data*, use `google_sheets_values_append` (**values**) instead — it auto-grows the grid.

## Capability cliffs — these live in OTHER categories

- **Freeze header rows / first columns** → **sheets** (`google_sheets_update_sheet_properties`, mask `gridProperties.frozenRowCount` / `frozenColumnCount`). Not a dimension op; don't bend `resize` to do it. This is the most common "make it professional" companion to resizing.
- **Hide / show rows or columns** → not here. `resize_dimension` only sets size (`fields` hardcoded to `pixelSize`). Hiding needs `updateDimensionProperties` + `hiddenByUser` via **raw** (`google_sheets_batch_update`).
- **Group / collapsible outlines** (+/− brackets) → **raw** only (`addDimensionGroup`).
- **Header styling, number formats, banding** → **formatting** (and `addBanding` via **raw**). This category never touches colors or fonts.
- **"Fit to one page" / print scaling / print area** → not exposed by any dimension tool. Resize sets pixel widths only; it cannot guarantee columns sum to one printed page. Set sensible widths as the best approximation and say page-fit isn't available here.

## Gotchas

- batchUpdate is atomic: one request per call, no partial success — a bad request fails the whole call.
- If you chain two calls, the second sees the post-mutation grid (delete rows 0–2, then "row 5" is now the old row 7). Re-resolve indices between chained mutations.
- `sheet_name` is case-sensitive (`"Sheet1"` ≠ `"sheet1"`); a miss raises a `ToolError` listing the real tab names. Prefer integer `sheet_id` from the metadata read.
