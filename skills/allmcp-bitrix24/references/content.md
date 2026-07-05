# Bitrix24 content — workflows

Four read-only list tools: Sites (websites + landing pages) and Document
Generator (documents + templates); creating, publishing, or editing happens
only in the Bitrix24 UI. Jump to the matching task.

## Model note — read before acting

- **Site and page links are unobtainable.** `PUBLIC_URL`/`DOMAIN_NAME` are
  calculated fields the connector never requests, and the public-URL endpoint
  is not wrapped. Report links as unavailable; never construct a URL from
  `CODE`, `DOMAIN_ID`, or anything else.
- **`include="full"` changes nothing in this category** — all four tools
  return identical payloads either way (the get_* tool their descriptions
  mention doesn't exist here). An absent key is truly absent — never re-fetch
  with `"full"` to recover it.
- **You only ever see live records.** Trashed sites/pages and deleted
  templates are excluded server-side — "not in the list" cannot distinguish
  trashed from never-existed; say so. Knowledge bases, workgroup sites, and
  the portal main page are also invisible to `bitrix24_list_sites`.
- `ACTIVE`/`DELETED` are `"Y"`/`"N"` strings, not booleans. `createdBy`/
  `CREATED_BY_ID` are numeric user IDs — resolve via `bitrix24_list_users`
  (`organization` category).

## Task: inventory sites — which do we have, which are live?

1. `bitrix24_list_sites()` → per item read `ID`, `TITLE`, `ACTIVE`
   (`"Y"` = published), and `TYPE`: `PAGE` = regular website, `STORE`/`SMN`
   = online store (storefront data lives in the `commerce` category).

## Task: list a site's pages / check whether a page still exists

1. `bitrix24_list_sites()` → match `TITLE` → site `ID`.
2. `bitrix24_list_landings(site_id=<step 1>)` → pages; each carries `ID`,
   `TITLE`, `ACTIVE` (`"Y"` = published), `SITE_ID`.
3. The `site_id` filter reaches the API in a shape it may ignore, returning
   every page on the portal — keep only items whose `SITE_ID` equals the
   step-1 `ID` before presenting anything as that site's page.
4. A page absent here may be trashed or never created — you cannot tell which.

## Task: find a generated document (contract / quote / invoice)

1. `bitrix24_list_documents(start=0)` → match client-side on `title`,
   `number`, `createTime`, or `value`. `value` joins the document to its
   entity, format `<OBJECT_TYPE>_<ID>`:
   `value="DEAL_215"` → `bitrix24_get_deal(deal_id=215)` (`crm` category).
2. Page by `total`, not `next_cursor` (often absent on this endpoint):
   repeat with `start += 50` while `start < total` and `items` is non-empty.
3. `downloadUrl` = DOCX, `pdfUrl` = PDF. PDF conversion is asynchronous and
   empty fields are stripped — a missing `pdfUrl` means no PDF yet *or*
   still converting; offer `downloadUrl` and suggest re-listing later.
4. Not found after paging to `total`? Documents generated from CRM entity
   cards (deal/quote UI) may live in a CRM-side document family these tools
   cannot see — report "may exist on the CRM side", not "doesn't exist".

## Task: list document templates / find the default

1. `bitrix24_list_templates()` → read `name`, `active`, `isDefault`.
2. If `items` is empty but `total > 0`, templates exist but the connector
   dropped them (response-shape coercion defect) — report the count and the
   defect, never "no templates".

## Pagination

Documents/templates: fixed 50 per page via `start`, top-level `total` (loop
above); page sequentially — deep paging costs against the ~2 req/s limit.
Sites/landings: `start` is not part of the landing contract — expect the full
live set in one response with `total`/`next_cursor` `None` (per docs).
