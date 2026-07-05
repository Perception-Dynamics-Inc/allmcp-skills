# Google Sheets developer metadata — workflows

Developer metadata is a key/value tag stamped on a spreadsheet, tab, row, or
column. Its point: the tag follows its data across row/column inserts, so you
re-find a region by tag after the A1 coordinates have drifted. Jump to the
matching task. Steps marked **✍** create, change, or delete tags — confirm a
vague request before firing one.

## Model note — read before acting

- **`location` takes an integer `sheetId`, never a tab name — and there is no
  resolver here.** Before any create on a tab/row/column, call
  `google_sheets_get_spreadsheet(spreadsheet_id, include="summary")`
  (**metadata** category) and read `sheets[].properties.{title, sheetId}` to map
  the tab name → integer `sheetId`. Never assume the target tab is `sheetId=0`;
  that holds only for the original first tab.
- **A row/column location anchors exactly one row or column** — a `dimensionRange`
  must have `endIndex` = `startIndex + 1`. A multi-row/-column span is rejected
  with an invalid-argument error. To mark a table, tag its header or first data
  row; that one row still drifts with inserts and re-finds the table.
- **`update`/`delete` always return `success=True`.** The real result is the
  count in the message (`"Updated N…"` / `"Deleted N…"`). N=0 means the filter
  matched nothing — treat that as a failure, not success. Read the count.
- **A key-only filter matches ALL records with that key.** Keys are not unique.
  The risk is touching *too many*, never too few. To act on exactly one record,
  filter by `metadataId`, not `metadataKey`.
- **This module never reads or writes cells.** Reading/updating the tagged cells
  lives in the **values** category (see Cross-category links). Don't promise to
  fetch the numbers from here.

## Task: mark a table so it survives row inserts, then read it by tag

The marker is a single row, not the whole block.

1. `google_sheets_get_spreadsheet(spreadsheet_id="1a2B…", include="summary")`
   → find the tab "Q3 Budget" → its integer `sheetId` (`gid`).
2. **✍** `google_sheets_create_developer_metadata(spreadsheet_id="1a2B…",
   metadata_key="invoices_table", location={"dimensionRange": {"sheetId": <gid>,
   "dimension": "ROWS", "startIndex": 11, "endIndex": 12}})` → returns the
   server-assigned `metadataId` as `id`; keep it. (`startIndex: 11` = row 12,
   0-based; `endIndex: 12` makes it exactly one row.)
3. Read the cells by tag — `google_sheets_values_batch_get_by_data_filter`
   (**values** category) with `data_filters=[{"developerMetadataLookup":
   {"metadataKey": "invoices_table"}}]`. This resolves the tag's *current*
   position, not the row 12 it started on.

## Task: change a tag's value or visibility

Touch only the fields you mean to change.

1. **✍** `google_sheets_update_developer_metadata(spreadsheet_id="1a2B…",
   data_filters=[{"developerMetadataLookup": {"metadataId": <id>}}],
   developer_metadata={"metadataValue": "totals_row", "visibility": "PROJECT"},
   fields="metadataValue,visibility")`. The `fields` mask is load-bearing: the
   default `"*"` overwrites *every* field from `developer_metadata`, blanking the
   key/location you didn't resend.
2. Confirm `N=1` in the message — not `success`. If `N=0`, the filter matched
   nothing.
3. Read back with `google_sheets_get_developer_metadata(spreadsheet_id="1a2B…",
   metadata_id=<id>)`. Use get-by-id, not `search`: setting `PROJECT` can hide
   the tag from a later `search` (ambiguous — hidden vs. never-changed), but
   get-by-id still resolves it.

## Task: find, then delete, tags by key

1. `google_sheets_search_developer_metadata(spreadsheet_id="1a2B…",
   data_filters=[{"developerMetadataLookup": {"metadataKey": "old_pipeline"}}])`
   → `items[]` (each a `DeveloperMetadata`), `total` = count. For *all* tags, use
   a location filter: `{"developerMetadataLookup": {"metadataLocation":
   {"spreadsheet": true}}}`.
2. **✍** `google_sheets_delete_developer_metadata(spreadsheet_id="1a2B…",
   data_filters=[{"developerMetadataLookup": {"metadataKey": "old_pipeline"}}])`.
   One key filter deletes every record with that key — do not fan out one filter
   per id "in case it deletes only one." To delete exactly one of several sharing
   a key, filter by its `metadataId`.
3. Confirm the count: `"Deleted 3…"`. `Deleted 0` means nothing matched (wrong
   spreadsheet, or the tags were `PROJECT`-scoped by another client) — report
   that, not success.

## Gotchas

- **`search` does not paginate** — no cursor, no `pageToken`; all matches return
  in one call. There is no list-all tool; broad-filter `search` is the substitute.
- **Tags don't appear in `get_spreadsheet`/list output** — those overviews strip
  `developerMetadata`. Only this category's tools surface tags; "no tags" from a
  spreadsheet read is wrong.
- **Drift covers inserts, not deletes.** A tag follows its row/column on *insert*;
  behavior when the tagged dimension is *deleted* is undocumented — re-resolve via
  `search` rather than promising the tag persists.
- **`PROJECT` tags from another client are invisible** to this connector's
  `search`/`get`. Default new tags to `DOCUMENT`; use `PROJECT` only for
  deliberate cross-client isolation.
- **404 on get/search means the connected account can't see this file** —
  re-share it with the connected Google account; never make the file public.

## Cross-category links

- **Resolve a tab name → integer `sheetId`** → `google_sheets_get_spreadsheet`
  (**metadata** category), `sheets[].properties.{title, sheetId}`.
- **Read or write the tagged cells by tag** →
  `google_sheets_values_batch_get_by_data_filter` /
  `google_sheets_values_batch_update_by_data_filter` (**values** category), same
  `{"developerMetadataLookup": {...}}` filter. This is the payoff for tagging.
