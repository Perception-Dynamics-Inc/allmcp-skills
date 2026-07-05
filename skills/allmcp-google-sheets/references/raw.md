# Google Sheets raw — workflows

`google_sheets_batch_update` is the only tool: it POSTs a list of raw
batchUpdate `Request` objects atomically. Reach for it ONLY when no convenience
tool expresses the request. Steps marked **✍** mutate the spreadsheet — confirm
specifics before firing one on a vague request.

## Model note — read before acting

- **Prefer the convenience tool; raw is the last resort.** Formatting, merges,
  data validation, conditional rules, charts, sorts/filters, named ranges,
  protection, find/replace, row/column resize, tab management — each has a
  dedicated tool (categories `formatting`, `conditional`, `charts`,
  `filters_sort`, `named_ranges`, `protection`, `find_replace`, `dimensions`,
  `sheets`, `tables`, `developer_metadata`) that resolves `sheetId` and field
  masks for you. Use raw only for what has no wrapper: `addBanding`, `pasteData`,
  `cutPaste`/`copyPaste`, `autoFill`, `textToColumns`, `addSlicer`,
  `insertRange`/`deleteRange`, dimension groups, data sources.
- **Raw resolves no names.** Each request body needs the integer `sheetId`,
  never a tab name. Resolve first via
  `google_sheets_get_spreadsheet(include="summary")` (`metadata` category) →
  `sheets[{sheetId, title}]`. `sheetId` 0 is a real, valid ID; never treat 0 as
  missing, and never assume the target tab is sheet 0.
- **Never probe with a write.** Do NOT freeze/rename/`findReplace` a guessed
  sheet just to read structure back. `findReplace` is a mutation (its reply
  reports `occurrencesChanged`), so `find==replacement` is NOT a safe no-op, and
  `allSheets:true` rewrites the whole workbook. Resolve via `metadata`; if only
  raw is reachable, ask the user for the `#gid=` in the tab's URL.
- **Raw cannot read cell values or verify extents.** batchUpdate is
  write/structure-only. To find which column "Region" is, how many rows hold
  data, or to confirm a write landed, use `google_sheets_values_get` /
  `values_batch_get` (`values` category). When you can neither read nor guess
  safely, stop and ask — never invent a column index or row count.
- **The batch is atomic.** One bad request rejects ALL of them; nothing applies
  and the error names `requests[i]`. No partial success. Isolate failure-prone
  `add*` requests (esp. `addBanding`, which rejects on overlap with an existing
  band) into their own call so a collision can't roll back safe formatting.

## Recipes

Resolve `SID` first (Model note). All ranges are `GridRange` (see Reference).

- **Polish a sheet (freeze + header + currency + zebra).** Batch the safe parts:
  `updateSheetProperties` (`gridProperties.frozenRowCount:1`), `repeatCell` (bold
  header), `repeatCell` (currency on the amount column — OMIT `endRowIndex` so it
  covers the whole column; a guessed row cap silently skips later rows). Then
  send `addBanding` as a SEPARATE ✍ call. Its body:
  `{"addBanding":{"bandedRange":{"range":{...},"rowProperties":{"firstBandColor":{"red":1.0,"green":1.0,"blue":1.0},"secondBandColor":{"red":0.95,"green":0.96,"blue":0.98}}}}}`.
- **Paste a CSV/HTML block.** ✍ `pasteData` with `coordinate:{sheetId,rowIndex,columnIndex}`
  (0-based; B2 = `{rowIndex:1,columnIndex:1}`), `type:"PASTE_NORMAL"`, and either
  `delimiter:","` or `html:true`. It overwrites covered cells. The reply is empty
  `{}` with no written-cell count — if the user wants the row count, read it back
  with `values_get`; don't count your own input.
- **Split a column ("City, State" → two columns).** `textToColumns` overwrites
  the columns to the RIGHT of the source. In ONE atomic ✍ batch, first
  `insertDimension` a blank column after the source, then `textToColumns`
  (`source` GridRange, `delimiter:", "`, `delimiterType:"CUSTOM"`). Confirm the
  source column via `values_get` or ask — do not guess column A.

## Capture new IDs

After any `add*`/`duplicate*` request, read the new ID from `replies[i]` (e.g.
`replies[0].addSheet.properties.sheetId`, `replies[0].addChart.chart.chartId`,
`replies[0].addBanding.bandedRange.bandedRangeId`). Never assume the index or ID
of a newly-created object. Most other requests return an empty `{}` reply.

## Gotchas

- **GridRange is 0-based, half-open: `endRowIndex`/`endColumnIndex` EXCLUSIVE.**
  Header row = `startRowIndex:0, endRowIndex:1`. Omit a bound = "whole
  column/row" — prefer this over a guessed extent.
- **Colors are floats 0.0–1.0, never 0–255.** `#1A73E8` → `{red:0.102,
  green:0.451, blue:0.910}`. 0–255 integers are silently wrong.
- **Use per-leaf `fields` masks on `updateCells`/`repeatCell`.** A broad mask
  like `userEnteredFormat.textFormat` replaces the whole sub-message and wipes
  unset siblings (font size, color). Mask the exact leaf: `...textFormat.bold`.
- **404 = the connected Google account can't see this file.** Have the user
  share it with that account; never advise making it public.
- **`requests` holds 1–1000 objects;** a huge single batch can also hit the
  ~2 MB / 180 s ceilings — split very large builds.

## Reference — NumberFormat patterns

`numberFormat:{type, pattern}` inside `userEnteredFormat`:

| `type` | `pattern` | Renders |
|---|---|---|
| `CURRENCY` | `"$#,##0.00"` | `$1,234.50` |
| `NUMBER` | `"#,##0"` | `1,235` |
| `PERCENT` | `"0.0%"` | `12.3%` |
| `DATE` | `"yyyy-mm-dd"` | `2026-06-18` |
| `DATE_TIME` | `"yyyy-mm-dd hh:mm"` | datetime |

Accounting red-negatives: `"$#,##0.00;[Red]($#,##0.00)"`.
