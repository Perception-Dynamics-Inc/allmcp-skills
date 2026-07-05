# Google Docs documents — workflows

This category reads Google Docs document content. Two tools, two approaches:

| Tool | Endpoint | Best for |
|---|---|---|
| `google_docs_get_document` | Docs API v1 | Title + text + word count in one call; handles most documents |
| `google_docs_export_as_text` | Drive export (`text/plain`) | Alternative when headers/footers/footnotes matter |

Both accept either a bare document ID **or** a full `docs.google.com` URL.
The tools extract the ID from the URL automatically.

## Model note — read first

- **Always prefer `get_document` for general reading.** It returns `{document_id, title, text, word_count}` and handles the common case: body paragraphs, lists, and inline tables (cells are joined with ` | `).
- **Use `export_as_text` as a fallback** when the user says the text looks wrong, or when the document has complex layouts (headers, footers, footnotes, text boxes). Drive's export engine handles these differently.
- **Images and drawings produce no text** — both tools silently skip them. If the user says "read the chart," inform them that visual elements cannot be read; only text content is available.
- **`word_count` (get_document) counts whitespace-split tokens**, not "words" in the linguistic sense. A 500-word doc might return `word_count=480` due to punctuation. Use it for rough sizing, not exact counts.
- **Return shapes**:
  - `get_document` → `{document_id, title, text, word_count}`
  - `export_as_text` → `{document_id, text, character_count}`

## Task: read a document the user shares as a URL

```
google_docs_get_document(
    document_id="https://docs.google.com/document/d/1BxiMVs0.../edit"
)
```

No need to list files first. The tool extracts the ID from the URL.

## Task: read a document by ID

```
google_docs_get_document(document_id="1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms")
```

## Task: user says "summarize this Google Doc" (link pasted in chat)

1. Call `google_docs_get_document(document_id=<URL or ID>)` → receive `text`.
2. Summarize the `text` directly. No second tool call needed.
3. If `get_document` returns an error about access, suggest the user share the
   doc with the connected Google account, then retry.

## Task: user wants the clean plain-text version (export)

```
google_docs_export_as_text(document_id="<ID or URL>")
```

Returns raw UTF-8 text as Drive's export engine produces it — no structure
parsing on AllMCP's side. Good fallback for documents with heavy formatting
where `get_document`'s text extraction feels off.

## Task: find a document by name, then read it

This is a two-step workflow spanning the `files` and `documents` categories:

1. `google_docs_list_files(name_contains="Project Brief")` → pick the correct
   `id` from the results (confirm with the user if multiple matches).
2. `google_docs_get_document(document_id=<id>)` → read the content.

## Gotchas

- **Empty document raises ToolError** — both tools raise `ToolError` if the
  document has no readable text. The document may genuinely be empty, or the
  connected account may lack read access to its content.
- **Tables are flattened** — `get_document` joins table cells with ` | ` and
  rows with `\n`. This is intentional; the AI agent can parse table structure
  from this format. If the user needs the raw table JSON, they should use the
  Docs API directly via the `raw` escape hatch (not yet implemented).
- **`text` may end with a newline** — Docs API always appends `\n` to each
  paragraph. Strip if needed before displaying.
- **Large documents** — `get_document` fetches the entire document in one
  call. Very large documents (100,000+ words) may result in large response
  payloads; consider `export_as_text` for those or summarize section-by-section.
- **Stale token → ToolError("expired or revoked")** — both tools raise this on
  a 401. The credential has been marked for reconnection; ask the user to
  reconnect their Google Docs account.
