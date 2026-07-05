# Bitrix24 lists — workflows

Universal lists are portal-defined registries — contract registries, supplier
directories, asset catalogs. These tools read the rows (elements) of one list
at a time; jump to the task matching the request. Everything here is read-only.

## Model note — read before acting

- **Read-only, elements only.** Nothing here adds, edits, or deletes an
  element, creates or configures a list, or downloads file attachments
  (file-type properties surface as bare numeric IDs). Don't promise any of
  these.
- **No list discovery.** Both tools require a numeric `iblock_id` and no tool
  can find one. Source it from the user or from the list's portal URL
  (`.../company/lists/<iblock_id>/...`). With neither, ask — don't probe IDs.
- **`iblock_type_id` is a closed three-value enum:** `lists` (standard lists,
  the default), `bitrix_processes` (processes run from the Feed),
  `lists_socnet` (workgroup lists — expected to fail here: the required
  `SOCNET_GROUP_ID` cannot be sent). Nothing else exists — never invent
  variants like `processes` or `lists_processes`.
- **Custom columns are anonymous.** They arrive as `PROPERTY_<id>` keys and
  their display names cannot be resolved here — infer meaning from the values
  or ask the user which property is which.
- **`include` is a no-op in this category:** `"summary"` and `"full"` return
  identical payloads on both tools, and a listed element already carries
  everything `bitrix24_get_element` would add. Only `efficiency_on=False` on
  the get tool changes the payload (raw, diagnostics only).

## Task: find a record / report over a registry

No server-side filter, sort, or column select exists — every search or
aggregation is a client-side scan.

1. `bitrix24_list_elements(iblock_id=52)` → match on `NAME` and `PROPERTY_*`
   values client-side. Read answers straight from the matched element — do
   NOT re-fetch it with `bitrix24_get_element`.
2. More pages: `next_cursor` → `start`, sequentially — parallel paging burns
   the same portal-wide ~2 req/s (standard-plan) budget and risks
   `QUERY_LIMIT_EXCEEDED`. Pages are fixed at 50; use page-1 `total` to size
   the job and warn the user before scanning a large registry
   (2,000 rows ≈ 40 calls).

## Task: look up a specific record by ID or portal URL

1. From a URL `.../company/lists/52/element/214/...` take `iblock_id=52`,
   `element_id=214`. If the element slot is `0`, don't fetch element 0 —
   confirm the real element ID with the user.
2. `bitrix24_get_element(iblock_id=52, element_id=214)` — add
   `iblock_type_id="bitrix_processes"` when the record is a Feed process.

## Gotchas

- **"Not found" is combination-keyed.** `bitrix24_get_element` matches
  `iblock_id` + `element_id` + `iblock_type_id` together, so a real element
  queried with the wrong list or type yields the same "Element X not found in
  list Y". On not-found, retry once with the other real type
  (`lists` ↔ `bitrix_processes`), then report the ambiguity (wrong list ID,
  wrong type, missing `lists` scope, or genuinely absent) instead of
  concluding the element was deleted.
- **`PROPERTY_<id>` values are dicts keyed by value-ID, not scalars:**
  `"PROPERTY_119": {"55": "Smith"}`. Take the dict's values (multiple entries
  = multi-value column); the numeric key is internal — never show it.
- `CREATED_BY` / `MODIFIED_BY` are numeric user IDs — resolve names via
  `bitrix24_search_users` in the `organization` category.
- Elements of a `bitrix_processes` list are driven by workflows — start or
  inspect those in the `automation` category; this category only reads them.
