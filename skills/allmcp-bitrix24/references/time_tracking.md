# Bitrix24 time tracking — workflows

Jump to the task matching the user's request. Steps marked **✍** change the
user's timesheet — if the request was vague, confirm before firing one.

## Model note — read before acting

- **Scope: status + plain clock in/out for the connected webhook owner at the
  current moment — nothing else.** Per-task worklog does not exist anywhere in
  this connector (not in the `tasks` category either) — say it's unsupported,
  don't redirect. Same for timesheet history beyond the current workday, and
  for other users' status: resolving their names via the `organization`
  category dead-ends, because no tool here accepts the resulting ID.
- **Verify a clock-in by `STATUS=OPENED`, never by `TIME_START` recency** —
  clocking in after an earlier same-day clock-out *resumes* that workday and
  keeps the original morning `TIME_START`.
- **`ACTIVE=false` is not a failure** — the workday change awaits supervisor
  confirmation. The clock action succeeded; report it as pending, don't retry.

## Task: clock in or out

1. **✍** `bitrix24_clock_in()` or `bitrix24_clock_out()`.
2. `bitrix24_get_time_status()` → confirm `STATUS=OPENED` (in) or `CLOSED`
   (out). After closing, report `DURATION` (time worked) and `TIME_LEAKS`
   (total break time), both `HH:MM:SS`.

## Task: report current status / hours worked today

1. `bitrix24_get_time_status()` → read `STATUS` (table below). An open day's
   payload lacks `TIME_FINISH` — nulls are stripped, not an error.
2. On an open day `DURATION` stays `00:00:00` — estimate elapsed time from
   `TIME_START` and report `TIME_LEAKS` (break time) alongside. `TZ_OFFSET`
   gives the user's timezone offset in seconds.

## Task: recover from a forgotten clock-out (`STATUS=EXPIRED`)

The connector cannot backdate the close — warn the user before acting.

1. `bitrix24_get_time_status()` → confirm `STATUS=EXPIRED`; relay
   `TIME_FINISH_DEFAULT` (Bitrix24's recommended close time, present only in
   this state). Choosing the close time isn't possible through this connector.
2. If the user still wants it closed now: **✍** `bitrix24_clock_out()`.
3. `bitrix24_get_time_status()` → report the `TIME_FINISH` actually recorded
   — never assert in advance what time Bitrix24 stamps.
4. If they are working today: **✍** `bitrix24_clock_in()`, then re-check
   `STATUS=OPENED`.

## Reference — `STATUS` values (fixed, not portal-configurable)

| `STATUS` | Meaning |
|---|---|
| `OPENED` | Workday active |
| `CLOSED` | Workday completed |
| `PAUSED` | On a break (no break tools in this connector) |
| `EXPIRED` | Opened on a previous day, never closed — see the recovery task |
