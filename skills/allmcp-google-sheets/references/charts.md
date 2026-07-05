# Google Sheets charts — workflows

`add_chart` / `update_chart` / `delete_chart` are pass-through wrappers — you
hand-author the whole `chart`/`spec` dict; nothing builds or defaults it. Jump to the
matching task. The table at the bottom holds the union field names you cannot guess.
**✍** marks a step that adds, replaces, or deletes a chart — confirm specifics on a
vague request before firing one.

## Model note — read before acting

- **Resolve the integer tab `sheetId` BEFORE any chart call.** Every `GridRange` source
  and `position.sheetId` is an integer gid, never a tab name. Gids are not sequential
  and the first tab is not guaranteed to be `0`, so never guess: call
  `google_sheets_get_spreadsheet(spreadsheet_id)` (metadata category) and map the tab
  title to its `sheetId`.
- **No read tool for charts here.** To get a chart's `chart_id` or its current spec,
  call `google_sheets_get_spreadsheet(spreadsheet_id, include="full")` (metadata) —
  charts live under `sheets[].charts[]` as `chartId` + `spec`. Default
  `include="summary"` drops charts, so `include="full"` is mandatory. Never
  update/delete on a guessed id — `delete_chart` removes whatever object holds that id.
- **To edit a chart, read its current spec first.** `update_chart` is a full
  replace: read the spec via `include="full"`, mutate that object, send the whole
  thing back — any field you omit reverts to default and silently drops.
- **No filter, move/resize, border, or slicer.** Use `google_sheets_batch_update`
  (`raw` category) for those. Charts reference existing cells only — write missing data
  first via `google_sheets_values_update` (values category).

## Task: add a chart of a range (column / line / pie / KPI)

Pick the union key by intent: `basicChart` for category comparison or a trend over
dates, `pieChart` for part-to-whole (≤ ~7 slices), `scorecardChart` for one big KPI
number. The chart-type enum and `basicChart.chartType` values are in the table below.

1. `google_sheets_get_spreadsheet(spreadsheet_id="<id>")` → the data tab's `sheetId`
   (call it `gid`).
2. **✍** `google_sheets_add_chart(spreadsheet_id="<id>", chart={spec, position})`. For
   `basicChart`: set `chartType`, plus `domains[].domain` (category) and
   `series[].series` (values), each a `sourceRange.sources[]` `GridRange` `{sheetId:
   gid, startRowIndex, endRowIndex, startColumnIndex, endColumnIndex}` — 0-based,
   end-exclusive; domain and every series share the same source count. For an
   overlay, the anchor cell is `position.overlayPosition.anchorCell` `{sheetId: gid,
   rowIndex, columnIndex}`. → returned `id` is the new `chartId`.

A `scorecardChart` aggregates the whole range — it cannot filter rows. For a
conditional KPI ("Q4 only"), do NOT `SUM` the full column: write the figure with a
formula (`SUMIFS`) into one cell via `google_sheets_values_update` (values), then point
`keyValueData` at that single cell with no aggregate.

## Task: change a chart's title/type, or delete it

1. `google_sheets_get_spreadsheet(spreadsheet_id="<id>", include="full")` → match the
   chart under `sheets[].charts[]` by title/position → its `chartId` (and `spec`).
2. To edit: mutate the retrieved `spec` (e.g. `title`, `basicChart.chartType`) keeping
   every other field → **✍** `google_sheets_update_chart(spreadsheet_id="<id>",
   chart_id=<step 1>, spec=<full mutated spec>)`. To remove: **✍**
   `google_sheets_delete_chart(spreadsheet_id="<id>", chart_id=<step 1>)`.
3. Confirm: re-read `google_sheets_get_spreadsheet(spreadsheet_id="<id>",
   include="full")` — the edited spec changed and no other field dropped, or the
   deleted `chartId` is gone from `sheets[].charts[]`. Neither tool returns the
   resulting spec, so this read is the only proof the write landed as intended.

For a professional look, author the spec directly — `title` + `titleTextFormat`,
`backgroundColorStyle`, axis `title`, deliberate `legendPosition`, per-series
`colorStyle` (one base color + one accent). Cell-level polish (number formats, header
fill, freeze, banding) is NOT in a ChartSpec — do it in the `formatting` / `sheets` /
`conditional` categories.

## Reference — `spec` chart-type union (set exactly one; no `...Spec` suffix)

| Union key | Use for |
|---|---|
| `basicChart` | bar / column / line / area / scatter / combo / stepped-area |
| `pieChart` | proportions / part-to-whole |
| `scorecardChart` | single KPI number (**not** `scorecardChartSpec`) |
| `waterfallChart` / `histogramChart` / `treemapChart` | variance bridge / distribution / nested proportions |
| `bubbleChart` / `candlestickChart` / `orgChart` | 3-var scatter / OHLC / hierarchy |

`basicChart.chartType`: `BAR` (horizontal), `COLUMN` (vertical), `LINE`, `AREA`,
`SCATTER`, `COMBO`, `STEPPED_AREA`. `legendPosition`: `..._LEGEND` (`BOTTOM`, `LEFT`,
`RIGHT`, `TOP`, `NO`; pie also `LABELED`). `aggregateType`: `SUM`, `AVERAGE`, `COUNT`,
`MAX`, `MIN`, `MEDIAN`.
