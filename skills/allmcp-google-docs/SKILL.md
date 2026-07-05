---
name: allmcp-google-docs
description: Read Google Docs through the AllMCP hub — list and search documents in Drive, read full document text, and export any doc as plain text from a link or file ID. Use when the user asks to find, read, summarize, or extract text from Google Docs via AllMCP. NOT for creating, editing, or sharing documents (this connector is read-only), or for calling the Google Docs API directly with your own credentials.
---

# Google Docs via AllMCP

Google Docs through AllMCP is a small, **read-only** connector: four
namespaced tools (`google_docs_list_files`, `google_docs_get_document`, …)
across two categories. It lists and searches Docs files in Drive, reads full
document text, and exports plain text — it **cannot create, edit, append to,
comment on, share, or delete documents**. Never promise writes. This file is
orientation and routing; the real playbooks are in `references/` — read the
matching one **before** chaining tools in a category you haven't used this
session.

Platform mechanics (endpoint, connecting, hidden tools, quota) live in the
`allmcp` skill — install it alongside this one.

## Connect

- Provider key: `google_docs` (exactly this, snake_case).
- Auth is **OAuth2** — call `connect_provider(provider_key="google_docs")`
  with nothing else. The response is `action_required` with a Google consent
  URL: **give that URL to the user and stop**. They sign in and approve in a
  browser, and the connection completes on its own.
- There are no API keys, tokens, or secrets to paste — never ask for any.
  AllMCP owns the consent flow and refreshes tokens in the background; the
  user only re-approves if they revoke access from their Google account.

## The three rules that prevent most failures

1. **A pasted link is enough.** Both content tools accept a full
   `docs.google.com` URL or a bare document ID and extract the ID
   themselves — when the user shares a link, skip the file listing entirely
   and read it directly.
2. **Access follows the connected Google account.** A 404 or an empty
   search result on a healthy connection usually means the document isn't
   shared with that account — ask the owner to share it, and never advise
   making a document public to work around access.
3. **Only text comes out.** Images, charts, and drawings yield no content;
   tables are flattened to text. If the user asks about a visual element,
   say up front that only surrounding text is readable.

## Categories → playbooks

Read the reference file before working in a category; each one carries the
return shapes, pagination, and the gotchas that aren't guessable from tool
descriptions.

| Category | Read | Covers |
|---|---|---|
| `files` | [references/files.md](references/files.md) | Drive-side layer: list/search Docs files, resolve names to IDs, file metadata (owner, timestamps, share link) |
| `documents` | [references/documents.md](references/documents.md) | Read full document text (`get_document`) and the plain-text export fallback (`export_as_text`) |

The common cross-category flow: resolve an ID with `files`, then read the
content with `documents` — unless the user pasted a URL, in which case go
straight to `documents`.

## Boundaries worth knowing up front

- **The entire surface is read-only.** No tool creates, updates, moves, or
  deletes anything — in Docs or in Drive. Route write requests elsewhere and
  say so plainly rather than attempting a workaround.
- Two ways to read content, one preference: `get_document` for general
  reading; `export_as_text` only as the fallback for documents with heavy
  headers/footers/footnotes. Details in the `documents` playbook.
- There is no whoami — nothing returns the connected account's own email.
  For "my document," disambiguate by name with the user.
