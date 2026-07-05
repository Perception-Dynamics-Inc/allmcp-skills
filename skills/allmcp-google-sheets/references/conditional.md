# Google Sheets conditional formatting — workflows

Jump to the task matching the request. These three tools manage value-driven highlighting on
one tab — boolean highlights and color-scale gradients. You pass the raw `rule` dict; this
category does **no** name→id or hex→color conversion. Steps marked **✍** change the user's
spreadsheet.

## Model note — read before acting

- **Rules are addressed only by positional `index`, and indexes shift.** No rule ID, no
  "list rules" tool here. `update`/`delete` reorder/delete whatever integer `index` you pass,
  with no check that it's the rule you meant; any add/delete renumbers every later rule. So
  before any `update`/`delete`, **read the live rules first** (recipe below) and derive the
  index from what you see — never carry a guessed or stale one.
- **`sheet_id` is the integer tab id** (not the name); it also appears in every
  `GridRange.sheetId` inside `rule.ranges`. Get it from `google_sheets_get_spreadsheet`
  (**metadata**), `sheets[].properties.sheetId`, or the tab URL `#gid=`.
- **Resolve column/row bounds from the data, never by guessing.** A wrong `sheetId` or
  column index silently formats the wrong tab/column and still returns success. Read row 1
  (via `google_sheets_get_spreadsheet`, or `google_sheets_values_get` in **values**) to find
  the target column's 0-based index and whether a header row exists, then size the range from
  the data, before writing — an off-by-one on the bounds silently mis-sizes the band.
- **Boundaries:** boolean rules style text + fill only. No number format, borders, or fonts
  (those are static, in **formatting**), and Sheets has **no data bars** (offer a color scale
  or `=SPARKLINE`). One rule per call; for several atomically, or for zebra-stripe banding
  (`addBanding`, a different feature), use `google_sheets_batch_update` (**raw**).

## Task: read existing rules + their indexes (do this before any update/delete/reorder)

1. `google_sheets_get_spreadsheet(spreadsheet_id="…", fields="sheets(properties(sheetId,title),conditionalFormats)")`
   → for the target tab: `properties.sheetId` (your integer `sheet_id`) and the
   `conditionalFormats` array, whose order **is** the index (element 0 = index 0).
2. Identify your target rule by inspecting each entry's `booleanRule`/`gradientRule`
   (its condition, range, or fill). Pass *that element's position* as `index`.

## Task: highlight cells past a threshold (e.g. red when stock < 10)

1. Read row 1 to confirm the column's 0-based index (e.g. "Stock" = column C = 2).
2. **✍** `google_sheets_add_conditional_format(spreadsheet_id="…", rule={"ranges": [{"sheetId": 0, "startRowIndex": 1, "endRowIndex": 200, "startColumnIndex": 2, "endColumnIndex": 3}], "booleanRule": {"condition": {"type": "NUMBER_LESS", "values": [{"userEnteredValue": "10"}]}, "format": {"backgroundColor": {"red": 0.96, "green": 0.80, "blue": 0.80}}}})`
   — `startRowIndex: 1` only if row 1 is a header; set `endColumnIndex` to the last data
   column from the header read.
3. **Read-back:** the tool returns `id="appended"`, **not** the real index. Re-run the
   read-rules Task to report the rule's actual position and confirm it landed.

## Task: heatmap a metric column (color scale)

Read intent past the literal words: small "Days Left" = urgent → red low, green high.

1. Read row 1 for the metric column's 0-based index.
2. **✍** `google_sheets_add_conditional_format(spreadsheet_id="…", rule={"ranges": [{"sheetId": 0, "startRowIndex": 1, "endRowIndex": 200, "startColumnIndex": 6, "endColumnIndex": 7}], "gradientRule": {"minpoint": {"color": {"red": 0.85, "green": 0.33, "blue": 0.31}, "type": "MIN"}, "midpoint": {"color": {"red": 1, "green": 0.90, "blue": 0.50}, "type": "PERCENTILE", "value": "50"}, "maxpoint": {"color": {"red": 0.34, "green": 0.73, "blue": 0.54}, "type": "MAX"}}})`
   - **`MIN`/`MAX` points must OMIT `value`; `NUMBER`/`PERCENT`/`PERCENTILE` must INCLUDE a
     numeric-string `value`.** Violating this is `INVALID_ARGUMENT`. The write is atomic, so
     the whole rule is rejected and nothing applies. `gradientRule` has no `format` field.
3. **Read-back** as above to report the index.

## Task: re-prioritize overlapping rules (lower index wins)

First matching rule wins; the API has no "Stop if true" flag, so order is the only control.

1. **Read the live rules first** (read-rules Task) and identify the *specific* and *generic*
   rules by their conditions — get each one's real index. Never assume which index is which.
2. **✍** `google_sheets_update_conditional_format(spreadsheet_id="…", sheet_id=0, index=<specific rule's real index>, new_index=0)` to lift it above the generic rule. Pass exactly one of `rule` (replace) or `new_index` (reorder), never both.
3. **Read-back:** re-read `conditionalFormats` and confirm the new order — every later index
   just shifted.

## Task: change or remove a rule

1. **Read the live rules first** for the target's real index and the tab's `sheet_id`.
2. Replace: **✍** `google_sheets_update_conditional_format(spreadsheet_id="…", sheet_id=0, index=<i>, rule=<full new rule dict>)` (a full rule, not a patch).
   Remove: **✍** `google_sheets_delete_conditional_format(spreadsheet_id="…", sheet_id=0, index=<i>)`.
3. **Read-back** to confirm — every later rule just renumbered.

## Gotchas

- A malformed `rule` 400s the whole atomic call (`INVALID_ARGUMENT`) — nothing applied; the
  error names the offending field path.
- Static fonts/borders/number formats → **formatting**; auto-size columns → **dimensions**;
  charts → **charts**.
