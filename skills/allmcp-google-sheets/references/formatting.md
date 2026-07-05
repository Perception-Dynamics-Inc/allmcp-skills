# Google Sheets formatting — workflows

This category is **unconditional per-cell look**: fill, font, alignment, number format, borders, merges, data-validation dropdowns. Jump to the matching task; the bottom table holds the enums you can't guess. Steps marked **✍** change the sheet. Ranges are half-open 0-based GridRanges (`start` inclusive, `end` exclusive); every tool takes `sheet_id` (int) OR `sheet_name` (case-sensitive tab title).

## Model note — read before acting

- **These are write-only tools — no "read current formatting", no extent detection.** Before bordering, merging, or striping a "table", resolve its real bounds: `google_sheets_get_spreadsheet` (**metadata**, `include="summary"`) gives sheetId + rowCount/columnCount, or read data via **values**. A wrong bound silently styles the wrong rectangle and still returns `success=True`.
- **Stay in the convenience tools.** `google_sheets_format_range` builds a granular field mask so omitted params preserve existing formatting — don't hand-roll `repeatCell`/`updateCells` in **raw** for a job these tools cover, or you wipe sibling format fields.
- **Capability cliffs — live in OTHER categories; never promise them from here:**
  - Alternating-row **banding / zebra stripes** → no tool in any category. Issue ONE `addBanding` request via **raw** (`google_sheets_batch_update`). Do NOT paint rows one-by-one with `format_range` — that won't auto-extend and burns N writes.
  - **Value-driven** color ("red if > 100", color scales, data bars) → **conditional**. `format_range` paints one flat color regardless of values.
  - **Frozen header, tab color, hide gridlines** → **sheets**. **Column width / auto-resize** → **dimensions** (`wrap_strategy="WRAP"` wraps text but does NOT widen). **Charts** → **charts**.
- **`MERGE_ALL` is destructive:** keeps only the top-left value, silently discards the rest. Title the cell first, then merge. **`set_data_validation(rule=None)` clears** validation — not a no-op.

## Task: make a report look professional (the polish pipeline)

Beautiful = real data, a styled header, restrained accents. Spans four categories, in order:

1. **values** — write REAL numbers with `USER_ENTERED` (store `1200.5`, never `"$1,200.50"` — pre-formatted strings break `=SUM`).
2. **metadata** — `google_sheets_get_spreadsheet(spreadsheet_id, include="summary")` → the tab's `sheetId`, `rowCount`, `columnCount`. These are your range bounds.
3. **✍** `google_sheets_format_range` — header row only `[0,1)` across the data columns: `bold=true, foreground_hex="#FFFFFF", background_hex="#2F4F4F", horizontal_align="CENTER"`.
4. **✍** `google_sheets_format_range` — each numeric column: `number_format_type` + `number_format_pattern` (see the enum table for patterns; mind the percent-stores-a-fraction trap).
5. **✍** `google_sheets_update_borders` — outline the table + underline the header; skip the inner grid (see borders task).
6. **sheets** — freeze the header row; optional tab color. **dimensions** — `google_sheets_auto_resize_dimension` to fit columns. **raw** — subtle zebra banding (`addBanding`, white + very-light grey).

Taste: one clean font, 10–11pt body; bold + larger for hierarchy; one base color + one accent; keep raw data and the formatted report on separate tabs.

## Task: header / number-format / dropdown / border (single writes)

Bounds come from the metadata read, not a guess; extend `end_column_index` to the table's real width.

- **Bold+center header:** **✍** `google_sheets_format_range(..., start_row_index=0, end_row_index=1, ..., bold=true, horizontal_align="CENTER")`.
- **Money/percent/date column:** **✍** `google_sheets_format_range(..., number_format_type=..., number_format_pattern=...)` — patterns in the enum table. Trap: PERCENT renders the stored fraction, so `0.0%` shows `0.156` as `15.6%`; if the cell holds `15.6` you get `1560.0%`.
- **Locked dropdown:** **✍** `google_sheets_set_data_validation(..., rule={"condition": {...}, "strict": true, "showCustomUi": true})` — see the condition-type table for the `condition` shape; `strict=true` rejects off-list input.
- **Outline + header underline:** **✍** `google_sheets_update_borders(..., top={"style": "SOLID"}, bottom=..., left=..., right=...)`, then a second call over `[0,1)` with `bottom={"style": "SOLID_MEDIUM"}`. To color a border, supply a raw `colorStyle.rgbColor` with **float 0.0–1.0** channels (mid-grey `{"red":0.6,"green":0.6,"blue":0.6}`) — borders take NO hex; only `format_range`'s `*_hex` params do.

## Task: title banner / reset

Both ops below are **lossy and silent** (MERGE_ALL drops every value but the top-left; clear_formatting resets all direct formatting). Each recipe ends with a read-back that emits a tool result — don't just trust `success=True`.

- **Banner:** **✍** write the title via **values**, then `google_sheets_merge_cells(..., merge_type="MERGE_ALL")`, then `google_sheets_format_range` (bold, larger, CENTER, fill). Then read the merged cell back — `google_sheets_values_get` over the banner range (**values**) — and confirm the surviving title is your title, not a neighbor's. `google_sheets_unmerge_cells` reverses the merge.
- **Strip formatting:** **✍** `google_sheets_clear_formatting(...)` resets ALL direct formatting to default (values kept) — can't un-bold one property; re-apply what you want. Then `google_sheets_get_spreadsheet(..., include="full")` (**metadata**) to confirm the cleared state before re-styling.

## Gotchas

- Ranges are half-open 0-based: rows 2–11 = `start=1, end=11`. The tool rejects `end<=start` but not a correct-but-wrong-row mistake — verify against the metadata/values read.
- To REMOVE a border send `{"style": "NONE"}`, not omission (omitting leaves the edge untouched).
- `sheet_name` is case-sensitive (`"dashboard"` ≠ `"Dashboard"`); prefer integer `sheet_id`.
- 404 means the connected Google account can't see the file → ask the user to share it with that account; never advise making it public.

## Reference — enums

**`number_format_pattern`**: money `$#,##0.00`; whole money `$#,##0`; percent 1dp `0.0%` (cell holds the fraction); accounting w/ red negatives `$#,##0.00;[Red]($#,##0.00)`; ISO date `yyyy-mm-dd`; grouped int `#,##0`.

**`merge_type`**: `MERGE_ALL` (one cell, keeps top-left — default), `MERGE_COLUMNS` (per column), `MERGE_ROWS` (per row).

**Border `style`**: `DOTTED` `DASHED` `SOLID` (thin) `SOLID_MEDIUM` `SOLID_THICK` `DOUBLE` `NONE` (clears the edge). Never send `STYLE_UNSPECIFIED`.

**`condition.type`** for `set_data_validation`: `ONE_OF_LIST` (dropdown from literal `values`), `ONE_OF_RANGE` (dropdown from a range, value `=Sheet1!A2:A100`), `NUMBER_BETWEEN`, `NUMBER_GREATER`, `TEXT_CONTAINS`, `TEXT_IS_EMAIL`, `DATE_BETWEEN`, `BOOLEAN` (checkbox), `CUSTOM_FORMULA`, `NOT_BLANK`.
