# Google Sheets sheets (tabs) — workflows

Manage a spreadsheet's **tabs** (Google calls them "sheets"): add, delete,
duplicate, rename, reorder, copy a tab to another file, and a general
`update_sheet_properties` escape hatch (freeze/hide/recolor/resize). Jump to the
matching task; the tables hold values you cannot guess. **✍** marks writes —
confirm a vague request first.

## Model note — read before acting

- **No list-tabs tool lives here.** To learn a tab's integer `sheetId`, its
  0-based `index`, or its exact (case-sensitive) title, call
  `google_sheets_get_spreadsheet(spreadsheet_id, include="summary")` in the
  **metadata** category. This is the mandatory first hop before any update that
  needs a `sheetId` or a position.
- **`sheetId` is the URL `gid`** (`…/edit#gid=123456789`) — an arbitrary stable
  number, NOT the tab's position and NOT sequential. Never guess `0/1/2`. The
  first tab's `sheetId` is often `0`, which is valid.
- **This category only manages tabs.** A new tab is an empty grid. Writing data
  needs **values**; cell fill/bold/borders/number-formats need **formatting**;
  column widths need **dimensions**; banding and charts need **raw** /
  **charts**. Don't promise cell styling, banding, or auto-fit from here.
- **Tab deletes are permanent** (no undo, no tab-level trash) and Google rejects
  deleting the **last remaining tab**. Confirm the exact name and that another
  tab survives first.
- **`update_sheet_properties` defaults to `fields="*"`, which is destructive** —
  `*` rewrites the WHOLE SheetProperties, so any property you omit from the dict
  (title, index, grid size, color, hidden) silently resets to its default. Always
  pass a narrow mask (`fields="hidden"`, `fields="gridProperties.frozenRowCount"`,
  `fields="tabColorStyle"`). Prefer `rename_sheet`/`reorder_sheet` — they set the
  mask for you; reach for the escape hatch only to freeze/hide/recolor/resize.

## Task: add a tab, then fill it

1. **✍** `google_sheets_add_sheet(spreadsheet_id="<id>", title="Q3")` → new
   `sheetId`. Add `index=N` to position it, `tab_color_hex="#1a73e8"` to color
   it (hex is fine here — `add_sheet` converts it).
2. Headers go in via **values**: `google_sheets_values_update(
   spreadsheet_id="<id>", range="Q3!A1", values=[["Item","Amount"]])` —
   reference the tab by **name**, not sheetId.

## Task: duplicate a template tab (the most common job)

For "a new month/client tab that keeps the formulas," duplicate — `add_sheet`
gives an empty grid.

1. To place it **after** an existing tab, first resolve that tab's 0-based
   `index` via `get_spreadsheet` (metadata) → say `June` is `2`. Never assume
   the named tab is at index 0.
2. **✍** `google_sheets_duplicate_sheet(spreadsheet_id="<id>",
   source_sheet_name="June", new_sheet_name="July", insert_sheet_index=3)`
   (`= June_index + 1`) → new `sheetId`.

## Task: make a report look professional (freeze / hide / recolor a tab)

Use `update_sheet_properties`, one call per property, each with its own narrow
`fields` mask (never the destructive `"*"` default — see Model note):

1. Resolve the tab's `sheetId` via `get_spreadsheet` (metadata) — this tool does
   NOT accept a tab name; `properties` must carry a literal integer `sheetId`.
2. Freeze the header — **✍** `google_sheets_update_sheet_properties(
   spreadsheet_id="<id>", properties={"sheetId":0,
   "gridProperties":{"frozenRowCount":1}},
   fields="gridProperties.frozenRowCount")`.
3. Hide a scratch tab — **✍** `…properties={"sheetId":12345,"hidden":true},
   fields="hidden"`.
4. Color "Final" green — **✍** `…properties={"sheetId":678,
   "tabColorStyle":{"rgbColor":{"red":0.0,"green":0.5,"blue":0.0}}},
   fields="tabColorStyle"`. Channels are floats **0.0–1.0** here (NOT 0–255, NOT
   hex, NOT the legacy `tabColor` key).

**Polish pipeline** for a full report spans categories — route to them, don't
promise them here: real numbers (not "$1,200" strings) via **values** →
number-formats + a bold white-on-dark header via **formatting** → freeze the
header (step 2) → auto-fit columns via **dimensions** → white/light-grey
**banding** via the **raw** `google_sheets_batch_update` `addBanding` request →
value-driven color via **conditional** → one chart via **charts**.

## Task: rename + reorder, copy cross-file, delete

- Rename: **✍** `google_sheets_rename_sheet(spreadsheet_id="<id>",
  sheet_name="Sheet1", new_title="Dashboard")`. Move to front: **✍**
  `google_sheets_reorder_sheet(spreadsheet_id="<id>", sheet_name="Dashboard",
  new_index=0)`.
- Copy into another workbook: **✍**
  `google_sheets_copy_sheet_to(source_spreadsheet_id="<src>",
  destination_spreadsheet_id="<dest>", sheet_name="Summary")` → returns the new
  `SheetProperties` (incl. its new `sheetId`). The tab lands as **"Copy of
  Summary"** at the **end** of the destination; there is no rename param here. To
  rename it, pass the returned `sheet_id` (not the old name) to `rename_sheet`.
  To *move* rather than copy, delete the source afterward.
- Delete: confirm exact (case-sensitive) names and that >1 tab survives via
  `get_spreadsheet` (metadata) first, then **✍**
  `google_sheets_delete_sheet(spreadsheet_id="<id>", sheet_id=2024001)` — prefer
  the integer `sheet_id` to dodge case-sensitivity.

## Reference — IDs & enums

No tool paginates; each is one call. Name-based calls cost one hidden metadata
read to resolve the name — pass the integer `sheet_id` to skip it.

**Tab color floats** (only inside a raw `update_sheet_properties` dict;
`add_sheet`'s `tab_color_hex` takes hex). Convert any `#rrggbb` by dividing each
channel by 255: green `{red:0.0,green:0.5,blue:0.0}`, red
`{red:0.8,green:0.0,blue:0.0}`, archive-grey `{red:0.6,green:0.6,blue:0.6}`.

**`fields` masks for `update_sheet_properties`** — pick the narrowest, comma-join
for several, never leave `"*"`:

| Intent | `fields` |
|---|---|
| Freeze rows / columns | `gridProperties.frozenRowCount` · `gridProperties.frozenColumnCount` |
| Hide / show tab | `hidden` |
| Recolor tab | `tabColorStyle` |
| Resize grid / hide gridlines | `gridProperties.rowCount,gridProperties.columnCount` · `gridProperties.hideGridlines` |

New-tab grid defaults: **1000 rows × 26 columns** (max 18278 columns).
