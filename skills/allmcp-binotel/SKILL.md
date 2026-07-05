---
name: allmcp-binotel
description: Work Binotel cloud telephony — call history, recordings, live calls, employee extensions, the built-in mini-CRM, click-to-call — through the AllMCP hub. Use when the user asks to analyze or act on their Binotel calls (pull a rep's week, fetch recordings, list missed calls, place or transfer a call, manage customer records) via AllMCP. NOT for sending SMS (Binotel tools send none), for telephony providers other than Binotel, or for calling Binotel's REST API directly with your own code.
---

# Binotel via AllMCP

Binotel is a cloud PBX widely used in Ukrainian sales organizations. Through
AllMCP your agent gets namespaced tools (`binotel_list_calls_for_employee`,
`binotel_get_recording_url`, …) organized into categories. This file is
orientation and routing; the real playbooks are in `references/` — read the
matching one **before** chaining tools in a category you haven't used this
session.

Platform mechanics (endpoint, connecting, hidden tools, quota) live in the
`allmcp` skill — install it alongside this one.

## Connect

- Provider key: `binotel` (exactly this).
- Credentials: a **key + secret** pair, passed as multi-field auth:
  `connect_provider(provider_key="binotel", extra_fields={"key": "...",
  "secret": "..."})`. The field names are exactly `key` and `secret`; both
  are required and case-sensitive.
- The pair is **not self-service** — there is nothing to copy from the
  cabinet UI. The cabinet owner emails `support@binotel.ua` from the address
  on file, subject `Получение авторизационных данных для REST API`;
  turnaround is about one business day. One key pair per cabinet (company).
- Each AllMCP user who needs Binotel connects with their own key + secret.
  These are live credentials — never echo the secret back or log it.
- On connect, `calls` and `stats` tools are advertised; the other three
  categories unlock via `describe_category("binotel", ...)` or automatically
  on first call.

## The three rules that prevent most failures

1. **A dial that "succeeded" hasn't reached anyone.** Every outbound dial
   tool returns only `{success, message}` — `success=True` means Binotel
   accepted the request, not that a phone rang or a human answered. The job
   is finished only after you pull the call row back (`calls`/`stats`) and
   read its `disposition`. Until then, report "dialing / accepted" — never
   "reached", "answered", or "delivered".
2. **History comes capped.** Per-employee history windows are hard-capped at
   **7 days** — a wider window is rejected outright, never clamped, so slice
   longer ranges into 7-day chunks yourself. Result lists truncate at 50
   rows with no paging: check `extra.truncated` before presenting any list
   or count as complete, and narrow the window until slices come back whole.
3. **Outbound dialing is real-world exposure.** These tools ring real
   phones — a wrong target rings a stranger. Confirm the exact number(s)
   with the user before any dial, bridge, or transfer. The tools carry a
   `confirm=True` gate, but choosing the right number is your judgment call,
   not the gate's.

## Categories → playbooks

Read the reference file before working in a category; each one carries the
unguessable enums (dispositions like `ANSWER` vs `NOANSWER`), the
extension/ID flow between tools, and the boundaries.

| Category | Read | Covers |
|---|---|---|
| `calls` | [references/calls.md](references/calls.md) | Point lookups by ID/phone/customer, recording URLs, live calls, today's missed |
| `stats` | [references/stats.md](references/stats.md) | Call history by employee, period, or day; polling since a timestamp; call-tracking |
| `employees` | [references/employees.md](references/employees.md) | Extension directory — the name→extension resolver for everything else — plus routes, voice files, presence |
| `customers` | [references/customers.md](references/customers.md) | Binotel's phone-keyed mini-CRM: search, create, update, labels, delete |
| `outbound` | [references/outbound.md](references/outbound.md) | Click-to-call, two-leg bridges, transfer, hangup, announcement and IVR dials |

## Boundaries worth knowing up front

- **No SMS.** Nothing in Binotel texts a customer — don't offer a text
  fallback after a missed call.
- **No conference calls.** A bridge is exactly two legs; there is no 3-way.
- **No creating or editing voice files or IVR scenarios.** Voice files can
  be listed but never uploaded or changed, and IVR scenario names can't be
  enumerated at all — both live in the Binotel cabinet portal.
- **No native CRM link.** Binotel stores only its own customer IDs and phone
  numbers — match to Bitrix24 / amoCRM / SalesDrive records by phone in that
  provider's connector.
- Recording URLs are signed links that expire in about **15 minutes** —
  fetch them last, right before handing off to download or transcription.
