---
name: allmcp-google-sheets
description: Work Google Sheets — read/write cells, manage tabs, format ranges, charts, filters, sharing, export — through the AllMCP hub. Use when the user asks to read or change anything in a Google spreadsheet (cell data, tabs, formatting, conditional rules, permissions, CSV/XLSX/PDF export) via AllMCP. NOT for calling the Google Sheets API directly with your own credentials, or for non-spreadsheet Drive work (folders, Docs, Slides).
---

# Google Sheets via AllMCP

Google Sheets through AllMCP is the full Sheets + Drive surface: namespaced
tools (`google_sheets_values_get`, `google_sheets_format_range`, …) organized
into categories. This file is orientation and routing; the real playbooks are
in `references/` — read the matching one **before** chaining tools in a
category you haven't used this session.

Platform mechanics (endpoint, connecting, hidden tools, quota) live in the
`allmcp` skill — install it alongside this one.

## Connect

- Provider key: `google_sheets` (exactly this, snake_case).
- Auth is **OAuth2** — call `connect_provider(provider_key="google_sheets")`
  with nothing else. The response is `action_required` plus a Google consent
  URL: give that URL to the user and stop. They approve in their browser and
  the connection completes on its own.
- Never ask the user for API keys, client secrets, or refresh tokens — there
  is nothing to paste. AllMCP owns the consent flow and refreshes tokens in
  the background.
- If calls that worked before start failing with auth errors ("needs
  reconnecting"), run `connect_provider` again — the user re-approves once;
  nothing else is lost.

## The three rules that prevent most failures

1. **IDs are resolved, never guessed.** The spreadsheet ID comes from the URL
   (`/spreadsheets/d/{ID}/edit`) or the `files` category — the Sheets API has
   no list of its own. The integer `sheetId` that tab-level and batchUpdate
   tools need comes from `google_sheets_get_spreadsheet` (`metadata`);
   `sheetId` 0 is a real ID, not "missing". In A1 ranges, quote tab names
   containing spaces (`'Q1 Sales'!A1:C10`) and match case exactly.
2. **`values_*` tools write content, never appearance.** Store real values
   (`1200`, `0.12`, real dates under `USER_ENTERED`), never display strings
   (`"$1,200.00"`) — those break `=SUM` and every number format applied
   later. Looks live in `formatting`, `dimensions`, `conditional`, `charts`,
   `tables`; prefer those convenience categories and keep raw
   `google_sheets_batch_update` as the last resort.
3. **A 404 on a valid connection means the connected Google account can't
   see that file** — wrong account or wrong ID, not a sharing-visibility
   bug. Have the user share the file with the connected account; never
   advise making it public.

## Categories → playbooks

Read the reference file before working in a category; each one carries the
ID-flow between tools, the write-verification steps, and the traps (ranges
that don't clear, half-open GridRange indexing, atomic batches).

| Category | Read | Covers |
|---|---|---|
| `files` | [references/files.md](references/files.md) | Drive-side list/search, create, copy, trash — the spreadsheet-ID resolver |
| `values` | [references/values.md](references/values.md) | Cell-range reads and writes, append, clear, batch variants |
| `metadata` | [references/metadata.md](references/metadata.md) | Spreadsheet structure reads — the `sheetId` and feature-ID resolver |
| `sheets` | [references/sheets.md](references/sheets.md) | Tab management: add, delete, duplicate, rename, reorder |
| `dimensions` | [references/dimensions.md](references/dimensions.md) | Insert, delete, resize, move, or auto-fit rows and columns |
| `formatting` | [references/formatting.md](references/formatting.md) | Cell formatting, merges, data validation |
| `conditional` | [references/conditional.md](references/conditional.md) | Highlight or color-scale cells by value |
| `filters_sort` | [references/filters_sort.md](references/filters_sort.md) | Basic filter, filter views, sort ranges |
| `find_replace` | [references/find_replace.md](references/find_replace.md) | Find/replace, trim whitespace, delete duplicates |
| `named_ranges` | [references/named_ranges.md](references/named_ranges.md) | Create, rename, repoint, or delete named ranges |
| `protection` | [references/protection.md](references/protection.md) | Protected ranges that restrict who can edit |
| `charts` | [references/charts.md](references/charts.md) | Add, update, or delete charts |
| `tables` | [references/tables.md](references/tables.md) | Structured tables (2024+ Sheets feature) |
| `developer_metadata` | [references/developer_metadata.md](references/developer_metadata.md) | Key-value annotations that survive row moves |
| `sharing` | [references/sharing.md](references/sharing.md) | Drive permissions and ACLs |
| `export` | [references/export.md](references/export.md) | Export to CSV / XLSX / PDF |
| `raw` | [references/raw.md](references/raw.md) | Raw batch_update escape hatch — only for what has no wrapper |

## Boundaries worth knowing up front

- `files` sees only spreadsheets: it cannot create, find, or move folders.
  New files land in My Drive root; `copy_spreadsheet(parent_folder_id=...)`
  needs a folder ID you already know.
- There is no whoami — nothing returns the connected account's email, so
  "my budget sheet" needs disambiguation with the user, not owner inference.
- A plain "delete" should mean trash (recoverable);
  `google_sheets_delete_spreadsheet` bypasses trash permanently — only on an
  explicit ask.
- Writes return success + cell counts, not a per-range result map — verify
  writes that matter by re-reading with `value_render_option="UNFORMATTED_VALUE"`.
- Google throttles around 60 reads + 60 writes per minute per user — use the
  `batch_*` tools, not per-cell loops.
- An empty `list_spreadsheets` can mean a narrow OAuth scope, not a missing
  file — don't declare a sheet gone on one empty result.
