# Google Sheets metadata — workflows

Both tools are READ-ONLY. This category is the structure-read hub the rest of
the connector leans on: `google_sheets_get_spreadsheet` resolves the integer
`sheetId` and the advanced-feature IDs that batchUpdate-backed categories need.
Jump to the task matching the request.

## Model note — read before acting

- **`include='full'` is the ONLY way to surface advanced-feature IDs** — named
  ranges, protected ranges, conditional formats, filter views, banded ranges,
  developer metadata. Summary strips them all. The `named_ranges`,
  `protection`, `conditional`, `filters_sort`, and `developer_metadata`
  categories all resolve their IDs from a `full` read here.
- **Metadata and cell values are separate axes.** `include='full'` returns
  metadata only; cells need `include_grid_data=True`, which is huge — prefer
  `google_sheets_values_get` (`values` category) for bounded reads.
- **`spreadsheet_id` comes from outside this category** — the URL, or
  `google_sheets_list_spreadsheets` / `google_sheets_get_file_metadata`
  (`files`). Neither tool here searches by file name.
- **Boundaries.** Nothing here renames the file, edits title/locale, adds or
  deletes tabs, or creates a range — those live in `files`, `sheets`, and the
  per-feature categories; `google_sheets_batch_update` (`raw`) covers the rest.
- **A 404 means the connected account can't see the file** (wrong account or
  ID). Never advise making it public — share it with the connected account.

## Task: list named / protected ranges, conditional formats, filter views

`google_sheets_get_spreadsheet(spreadsheet_id="<id>", include='full')` → read
`namedRanges[]`, or per-tab `sheets[].protectedRanges[]` / `conditionalFormats[]`
/ `filterViews[]` / `bandedRanges[]`. Hand the resolved id to the owning
category to edit it. Pass `efficiency_on=False` only if a downstream tool needs
a field this strips as noise.

## Task: fetch the block tagged with developer metadata (rows may have moved)

When the user references a region by a metadata tag (e.g. `invoices_2026`)
rather than coordinates, target it with a `developerMetadataLookup` DataFilter
— coordinate-independent, survives row moves.

1. `google_sheets_get_by_data_filter(spreadsheet_id="<id>",
   data_filters=[{"developerMetadataLookup": {"metadataKey": "invoices_2026"}}],
   include_grid_data=True)`.
2. The match is broad: `locationMatchingStrategy` defaults to
   `INTERSECTING_LOCATION`, which returns every location *intersecting* the
   tag's, not just the tagged range. When the user says "just that range," add
   `"locationMatchingStrategy": "EXACT_LOCATION"`.
3. The response is RAW and unbounded — no lean mode, no `include`/
   `efficiency_on` knob. After the fetch, read the matched metadata's
   `metadataLocation` and narrow client-side to its `dimensionRange` rather
   than assuming the whole payload equals the tagged block. Tags are created in
   `developer_metadata`.

## Gotchas

- **`get_by_data_filter` has no lean mode** — use `get_spreadsheet` summary for
  plain structure discovery; reach for the filter tool only for metadata-tag or
  precise-coordinate targeting.
- **No pagination.** Both tools return the whole (filtered) resource at once;
  for a huge file restrict with `ranges` + `fields` instead of paging.

## Reference — DataFilter shapes

Each filter in `data_filters` must set **exactly one** of these; setting two is
invalid.

| Filter field | Shape | Targets |
|---|---|---|
| `a1Range` | string, e.g. `"Sheet1!A1:B10"` — 1-based, end-inclusive | that A1 range |
| `gridRange` | `{sheetId:int, startRowIndex, endRowIndex, startColumnIndex, endColumnIndex}` — **0-based, half-open** | that grid range |
| `developerMetadataLookup` | `{metadataId, metadataKey, metadataValue, locationType, visibility, metadataLocation, locationMatchingStrategy}` — set only the fields to match on | metadata-tagged ranges |

`locationMatchingStrategy`: `INTERSECTING_LOCATION` (default, broad) or
`EXACT_LOCATION`. Don't mix the A1 (1-based, inclusive) and gridRange (0-based,
half-open) models.
