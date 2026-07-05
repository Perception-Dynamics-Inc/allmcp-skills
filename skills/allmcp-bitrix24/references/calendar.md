# Bitrix24 calendar — workflows

Jump to the task matching the user's request. Steps marked **✍** write to a
calendar — if the request was vague, confirm date, time, and owner first.

## Model note — read before acting

- **Create + read only, solo events only.** No update, delete, attendees,
  recurrence, reminders, or all-day flag; group/company calendars unreachable.
- **"Current user" = the webhook owner**, not necessarily the person talking.
  Resolve every person — including the speaker — to a numeric user ID via
  `bitrix24_search_users` / `bitrix24_list_users` (`organization` category);
  never scrape IDs off event fields or ask the human before trying those.
- **Never parse `DATE_FROM`/`DATE_TO` for time math.** They are portal-locale
  display strings (`"12/11/2024 05:59:00 pm"` — day/month order unknowable).
  Overlap, duration, and free/busy arithmetic needs `include="full"` and the
  unix-seconds `DATE_FROM_TS_UTC`/`DATE_TO_TS_UTC` (zone: `TZ_OFFSET_FROM`,
  offset in seconds).
- Logging a client meeting on a CRM record is a CRM activity (`crm` category),
  not a calendar event; task deadlines live in `tasks`.

## Task: what's on a calendar / is someone free at a given time?

1. `bitrix24_search_users(query="Marina")` → user `ID`.
2. `bitrix24_list_events(owner_id=<step 1>, from_date="2026-06-18",
   to_date="2026-06-19")` — summary suits a plain agenda; use `include="full"`
   when you will compute overlap or duration.
3. For free/busy compare against `DATE_FROM_TS_UTC`–`DATE_TO_TS_UTC`; a row
   with `RRULE` is a recurring series — confirm the date falls in it first.
   The listing may omit meetings the person was merely invited to (unverified)
   — present "free" verdicts as best-effort, never a guarantee.

## Task: book an event at a given time

1. Resolve the owner's numeric user ID as above (even for "me").
2. **✍** `bitrix24_create_event(name="Budget review", owner_id=<step 1>,
   from_datetime="2026-06-18T15:00:00+02:00",
   to_datetime="2026-06-18T15:30:00+02:00")` → event `id`.
3. `bitrix24_get_event(event_id=<step 2>)` → check `DATE_FROM_TS_UTC` is the
   intended instant. A wrong-zone booking raises no error — this read-back is
   the only thing that catches it.

Plain datetimes land in the **webhook user's** timezone (no timezone param) —
embed the UTC offset in the ISO string when booking on someone else's calendar.

## Task: who's invited, and did they accept?

`bitrix24_get_event(event_id=181)` → `ATTENDEE_LIST`
statuses: `Y` accepted · `N` declined · `Q` no answer yet · `H` organizer.
Summary mode usually carries no attendee data — never answer RSVP from it.

## Task: reschedule or cancel ("move the 2pm meeting")

No update or delete exists — recreate: `bitrix24_get_event` → copy `NAME`, take
duration from `*_TS_UTC` → **✍** `bitrix24_create_event` at the new time → read
back the new id as in "book an event" → tell the user to delete the old one in
the Bitrix24 UI (the copy has no attendees).

## Gotchas

- No paging and no truncation — the whole window returns in one response; bound
  payload size only by narrowing `from_date`/`to_date`. Omitting dates silently
  covers just 1 month back → 3 months ahead — sweep monthly for older events.
- Omit `section_id` on create — Bitrix auto-detects the owner's default
  calendar. No tool lists sections, so never invent or ask for an ID.
