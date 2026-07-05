# Google Docs files — workflows

This category is the Drive-side file layer: list/search Google Docs files
and get file-level metadata (name, owner, timestamps, share link). Almost
every task starts here — you **resolve a document ID** from a name search or
a URL, then read the content with the `documents` category tools.

## Model note — read first

- **An empty `list_files` result is ambiguous, not proof of absence.** With
  `drive.metadata.readonly` scope, Drive returns HTTP 200 with an empty list
  even when matching docs exist if they belong to an account the OAuth app
  hasn't been granted access to. Before reporting "no such file," say
  *nothing visible to this account* and suggest the user share the document
  with the connected Google account — do not declare it gone.
- **`get_file_metadata` returns Drive metadata, not content** — name, owner,
  timestamps, `webViewLink`, parent folder IDs. For document text use
  `google_docs_get_document` or `google_docs_export_as_text` in the
  `documents` category.
- **There is no whoami tool.** Each file carries `ownerEmail`, but nothing
  returns the connected account's own address. For "my document," disambiguate
  by name with the user; never assume `items[0]` is the one they mean.
- **Return shapes** (don't guess fields): list rows and `get` with
  `include="summary"` (default) =
  `{id, name, mimeType, modifiedTime, createdTime, ownerEmail, webViewLink, parents}`;
  `get` with `include="full"` = raw Drive payload with `owners` array, `shared`
  boolean, and other Drive fields (`ownerEmail` is NOT present — use
  `owners[0].emailAddress` instead).

## Task: find a document by name (resolve its ID)

`google_docs_list_files(name_contains="Q3 report")` → scan rows, pick the one
the user means → `id`. `name_contains` is a case-insensitive substring on the
file name only; omit it to list all, but `""` / whitespace is rejected. On an
empty result apply the scope caveat above before reporting the document missing.

## Task: open a document the user shared as a URL

The user pastes `https://docs.google.com/document/d/1BxiMVs0.../edit`.

Skip the file listing step entirely — pass the URL directly to
`google_docs_get_document(document_id="https://docs.google.com/document/d/...")`.
The tool extracts the ID from the URL automatically.

## Task: inspect who owns a document and when it was last edited

`google_docs_get_file_metadata(file_id="<id>")` → default `include="summary"`
returns `ownerEmail`, `modifiedTime`, `createdTime`, `webViewLink`, and `parents`
(owner data is flattened). Use `include="full"` to receive the raw Drive payload
including `owners[0].emailAddress`, `shared`, and other Drive fields. This is
Drive metadata — it does not include the document's text.

## Pagination

Only `list_files` paginates. `total` is always `None` — Drive reports no
count. Pass the prior call's `next_cursor` as `page_token` and repeat until
`next_cursor` is null. Default `page_size` is 100 (max 1000).

## Gotchas

- **404 with a valid connection means the connected Google account can't see
  the file** — wrong account or wrong ID. Have the user share the document
  with the connected account; **never advise making it public.**
- **`include_trashed=True`** is needed to see documents the user moved to
  trash. By default trashed files are excluded.
- **`shared_with_me_only=True`** narrows to documents shared *with* the
  connected account (not owned by it). Useful when the user says "the doc
  my colleague shared."
- **`modifiedTime desc`** (the sort default) is not "created most recently."
  A recently edited old doc sorts to the top; don't read `items[0]` as "the
  doc the user means."
