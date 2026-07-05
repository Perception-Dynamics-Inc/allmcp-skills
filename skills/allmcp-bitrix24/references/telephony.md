# Bitrix24 telephony — workflows

Two tools: `bitrix24_list_calls` (history) and `bitrix24_register_call` (log
the start of a call made outside Bitrix24). The enum tables at the bottom drive
every classification. **✍** marks writes — confirm vague requests first.

## Model note — read before acting

- **Read-mostly surface.** No dialing, no recording attach/upload, no
  lines/SIP management, no server-side history filters. Recordings are
  read-only via `CALL_RECORD_URL` (empty = no recording);
  `TRANSCRIPT_PENDING == "Y"` means the *transcript* is pending — it says
  nothing about the recording itself.
- **Register starts a lifecycle we can never finish** — Bitrix24 finalizes a
  call (statistics row + CRM activity) only via an unwrapped finish step.
  Never claim the call is now on the CRM timeline; report it as registered.
- **Registering is user-visible:** it pops a live call card on the attributed
  employee's Bitrix24 screen. Say so before firing.
- **Always pass `user_id_b24`** — the API documents USER_ID or
  USER_PHONE_INNER as required on register. Resolve the rep via
  `bitrix24_search_users` (organization category). Register never
  auto-creates leads.
- Identify callers via the row's `CRM_ENTITY_TYPE` + `CRM_ENTITY_ID` →
  `bitrix24_get_contact`/`bitrix24_get_lead`/`bitrix24_get_company`;
  `CRM_ACTIVITY_ID` → `bitrix24_get_activity` (crm category); `PORTAL_USER_ID`
  → organization category. No match? Search crm by exact `PHONE_NUMBER` only.

## Task: call volume, missed calls, or per-rep performance report

1. `bitrix24_list_calls(start=0)` → check `CALL_START_DATE` (ISO-8601 with
   timezone) across the first page before assuming newest-first; the default
   sort is not guaranteed. Early-stop a date-bounded scan only after
   confirming order.
2. Page sequentially, never parallel: the whole Bitrix24 account shares the
   rate budget (~2 req/s on standard plans). Bound the scan to the user's
   date range or the loop hits `QUERY_LIMIT_EXCEEDED`.
3. Classify via the tables below: incoming = `CALL_TYPE` {2,3}, rep-initiated
   = {1,4}; missed = `CALL_FAILED_CODE == "304"` exactly — never use
   `CALL_DURATION` (seconds) `== 0` as a proxy; declined/busy/invalid also sit
   at zero. Group reps by `PORTAL_USER_ID`; `CALL_VOTE` rates 1–5 (0 = unrated).

## Task: log a call made outside Bitrix24

1. `bitrix24_search_users(query="Dmitri")` (organization) → user `ID` 42.
2. **✍** `bitrix24_register_call(phone_number="+15551234567", call_type=1,
   user_id_b24=42)` → `CALL_ID` (string).
3. Report it as registered, not finalized (see Model note), and mention the
   call-card popup. Success is the returned `CALL_ID` — do not verify via
   `bitrix24_list_calls`; an unfinished call may not appear there.

## Gotchas

- `include` is a no-op here — summary and full return identical rows; its
  "get_* with efficiency_on=False" escape hatch does not exist in telephony.
- Re-registering the same number + type + user within 30 minutes may return
  the existing `CALL_ID` instead of a new one — not a duplicate-record bug.
- `insufficient_scope` means the webhook was minted without the telephony
  scope — re-issue the webhook; retrying won't help. `ACCESS_DENIED` has
  other documented causes (REST API unavailable on free plans; register may
  be refused outside app context) — also not fixed by retrying.

## Reference — call enums

**`CALL_TYPE`** (also register's `call_type`; all five values are valid even
though the param description lists two): `1` Outgoing, `2` Incoming,
`3` Incoming with redirection, `4` Callback, `5` Informational.

**`CALL_FAILED_CODE`** (outcome): `200` success, `304` missed, `603` declined,
`603-S` canceled, `486` busy, `404` invalid number, `480` temporarily
unavailable, `484`/`503` unavailable, `403` forbidden, `402` insufficient
funds, `423` blocked, `OTHER` undefined.
