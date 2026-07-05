# Google Sheets protection — workflows

Three tools, all writes: add, update, delete protected ranges. There is **no
list/get tool here** — discovering or auditing protections happens in the
`metadata` category. Jump to the matching task; steps marked **✍** change the
user's spreadsheet, so confirm the range and editors on a vague request first.

## Model note — read before acting

- **`range.sheetId` is the integer tab id, not the tab name — and this module
  does NOT resolve it for you.** Unlike `named_ranges`, `protected_range` goes to
  Google verbatim. Never guess `sheetId: 0`; resolve the tab name to its integer
  `sheetId` via `metadata` first. A wrong id silently protects the wrong tab.
- **Update with the default `fields="*"` is a full replace** — it clears every
  field you don't re-send, including `range`. To flip one field, name only it in
  the mask (e.g. `fields="warningOnly"`).
- **Owner and the connected user always keep edit access** — you can't lock them
  out. Describe a hard protection as "everyone except the editor list + owner +
  you," not a total lockdown.
- **`warningOnly: true` silences `editors`** — anyone may edit after a confirm
  prompt. Pick one mode, never both.
- **Protection-editor ≠ file access.** `editors.users` doesn't let them open the
  file; pair with `google_sheets_share_spreadsheet` (`sharing`) if needed.
- Set `range` or `namedRangeId` but **never both** — Google rejects the both-set
  write. To back a protection with a named range, create it in `named_ranges`
  first and pass its `namedRangeId` instead of a `range`.

## Task: lock a header/total row, or protect a whole tab

GridRange bounds are 0-based, half-open (row 1 = `startRowIndex=0,
endRowIndex=1`). A `range` with only `sheetId` protects the whole tab.

1. `google_sheets_get_spreadsheet(spreadsheet_id="<id>", include="full")`
   (`metadata`) → the target tab's integer `sheetId`.
2. **✍** `google_sheets_add_protected_range(spreadsheet_id="<id>",
   protected_range={"range": {"sheetId": <step 1>, "startRowIndex": 0,
   "endRowIndex": 1}, "description": "Header — finance only", "editors":
   {"users": ["finance@acme.com"]}})` → `id` (the `protectedRangeId`).

For a speed-bump not a wall, drop `editors` and set `"warningOnly": true`. To
lock a whole tab but keep input cells editable, use `range={"sheetId": <step
1>}` plus `"unprotectedRanges": [<GridRange>]` — `unprotectedRanges` is valid
ONLY on whole-tab protections.

## Task: change editors / flip / remove an existing protection

Update and delete need the `protectedRangeId`, which only `metadata` surfaces.

1. `google_sheets_get_spreadsheet(spreadsheet_id="<id>", include="full")` → read
   `protectedRanges` → the target's `protectedRangeId`.
2. **✍** Change editors (narrow mask — anything omitted under `fields="*"` is
   wiped): `google_sheets_update_protected_range(spreadsheet_id="<id>",
   protected_range={"protectedRangeId": <step 1>, "editors": {"users":
   ["finance@acme.com", "ceo@acme.com"]}}, fields="editors")`.
   Flip to warning-only: `protected_range={"protectedRangeId": <step 1>,
   "warningOnly": true}` with `fields="warningOnly"`.
   Remove entirely: **✍** `google_sheets_delete_protected_range(
   spreadsheet_id="<id>", protected_range_id=<step 1>)`.
3. After any update, re-read with `google_sheets_get_spreadsheet(include="full")`
   and confirm the protection's `range`/`namedRangeId` and the fields you meant to
   keep survived — the update tool reports `success` without reading back, so a
   too-wide mask that wiped the range looks like success here.

If `protectedRanges` is absent under `include="full"`, the tab truly has none —
but a default `include="summary"` read omits them, so never conclude "nothing is
protected" from a summary read.

## Gotchas

- **No read tool here.** Every update/delete/audit starts with
  `google_sheets_get_spreadsheet(include="full")` in `metadata`; the default
  `include="summary"` strips `protectedRanges` entirely.
- `batchUpdate` is atomic — a malformed `protected_range` rejects the whole call.
- A 404 on a valid token means the connected Google account can't see the file —
  share it with that account; never make it public.
