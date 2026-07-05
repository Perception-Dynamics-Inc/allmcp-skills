# SalesDrive categories — workflows

Two write tools — `salesdrive_upsert_categories` (create + update) and
`salesdrive_delete_categories`. Jump to the matching task; **✍** marks a
live-catalogue change, so confirm specifics on a vague request first.

## Model note — read before acting

- **Write-only category.** No list, get, count, or search tool for categories
  exists anywhere in the connector — you cannot show, count, or check existence
  of categories; say so plainly rather than guess.
- **Every op keys on the category `id`, and you can't obtain one here.**
  Upsert reports success but never returns the new id, and no read tool exists;
  valid ids come only from the SalesDrive web UI (catalogue) or YML export. So
  **never create a category meaning to fix it later** — its id is then
  unrecoverable, so you can't rename, reparent, nest under, or delete it until
  the user reads that id externally. There is no rename/move/delete-by-name.
- These two tools only read/write `{id, name, parentId}` per category — there's
  no field to set a description, image, sort-order, or active/archive flag, and
  no reorder or soft-delete/undo. (Whether SalesDrive's own category model has
  more is unknown here; the connector just doesn't expose it.)
- Filing a product *into* a category is a `products`-category job
  (`salesdrive_upsert_products`, `category` `{id, name}` field) — and still needs
  the category id from the UI/feed.

## Task: add categories (top-level, nested, or a bulk tree)

1. Top-level: **✍** `salesdrive_upsert_categories([{"name": "Boots"}])` (omit `parentId`).
2. Nested: get the parent's id from the SalesDrive UI / YML export, then **✍**
   `salesdrive_upsert_categories([{"name": "Winter boots", "parentId": "10"}])`.
3. Bulk tree: **✍** chain calls for a tree larger than one batch. Top-levels
   import blind, but nesting children needs each parent's id (upsert won't return
   it) — supply parent ids up front, or import parents, have the user read their
   ids from the UI / export, then upsert the children.

## Task: rename, reparent, or delete a category

All three key on the category's own id — source it from the SalesDrive UI / YML
export (not lookupable here).

- Rename / reparent (update): **✍** `salesdrive_upsert_categories([{"id": "42", "name": "Winter footwear", "parentId": "15"}])`. Moving back to top level isn't possible — `parentId: null` is silently dropped (see Gotchas); do it in the UI.
- Delete: **✍** `salesdrive_delete_categories(["88", "91", "80"])`. You can't confirm a category is empty here (no read tool), so verify in the UI first if "delete only if empty" matters.

## Gotchas

- **Omitting a field, or passing it as `null`, leaves it unchanged** — the
  connector strips nulls before sending, so you can't un-nest a category or clear
  a field; an omitted/null value keeps the server's existing one. (So a rename
  can safely omit `parentId`, and a reparent can omit `name`.) To un-nest (move
  to top level), do it in the SalesDrive UI.
