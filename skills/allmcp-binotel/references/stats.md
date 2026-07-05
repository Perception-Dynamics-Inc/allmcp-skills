# Binotel stats — finish the manager's question

The manager doesn't want call rows — she wants an answer from the phone logs: *how
did Marina do this week*, *how many did we miss today vs pick up*, *what's landed
since my last sync*. These tools hand you raw rows; the finished job is the count
you compute from them — split answered-vs-missed correctly, never a silent 50-row
slice reported as the whole, with recordings pulled on request. Nothing here
mutates; calls are events. Match the job below.

## "How did Marina do this week?"

The manager names a person and a week. You turn that into her extension, a legal
window, and a talked-to-someone count.

1. `binotel_list_employees()` (employees category) → find Marina → her row carries
   `extensionNumbers: ["0901"]`. Carry that extension as-is — the string `"0901"`.
2. Turn "this week" into UTC seconds. Monday 2026-06-29 00:00 **Kyiv** (the cabinet
   thinks in Kyiv, UTC+3 in summer; the API takes UTC) → `start_time=1782680400`;
   now, Wed 12:00 Kyiv → `stop_time=1782896400`. That's a 60h window, safely inside
   the hard **≤ 7-day** cap on this tool (longer is rejected, not chunked — loop
   7-day sub-windows yourself).
   `binotel_list_calls_for_employee(internal_number="0901", start_time=1782680400,
   stop_time=1782896400)`. Leave `include` at its `summary` default — the row
   already gives you `callType`, `disposition`, `billsec` and `startTime`; `full`
   only un-drops raw provider noise and buries the fields you're counting.
3. **The split that makes her number correct is client-side, so an invented token
   silently matches zero and you'll report she had no answered calls.** "Actually
   talked to someone" = `disposition == "ANSWER"` with `billsec > 0` — the value is
   `ANSWER`, never `ANSWERED`. Keep this a live-human count: `VM`/`VM-SUCCESS` (a
   voicemail she left, not a conversation) and `SUCCESS` (generic) are deliberately
   out — they belong to the wider recording-eligible set, not "she talked to them."
   Missed = `{NOANSWER, BUSY, CANCEL, CONGESTION, CHANUNAVAIL}` — `NOANSWER` is one
   word, there is no `MISSED` or `"NO ANSWER"`.
4. 📊 **Before you report a count, read `total` and `extra.truncated` — 50 rows back
   is NOT proof there were 50.** A busy week of 300 calls comes back as exactly 50
   with `truncated=True`. There is no paging (`next_cursor` is always `None`), no
   bigger-limit tool, and switching direction or to the period tool does not lift
   the ceiling — the 50 cap is ours, applied to every stats tool regardless of
   Binotel's own record caps. The only true count comes from **narrowing the window
   into sub-slices that each return under 50, then summing.**

## "Pull the recordings for the calls she answered"

The rows have no audio field, and `include="full"` will not conjure a recording
link — it only un-drops noise. **The MP3 lives one category over**, fetched per row
by `generalCallID` from step 3's result.

1. Keep only `disposition ∈ {ANSWER, VM, VM-SUCCESS, SUCCESS}` — the recording-
   eligible set. Everything else has no audio; don't fetch it.
2. Per kept row: a row like `{generalCallID:"1000000123456", disposition:"ANSWER",
   billsec:214}` →
   `binotel_get_recording_url(general_call_id="1000000123456", disposition="ANSWER")`
   (calls category) → a signed MP3 `url` with `expires_at` ~15 minutes out. That TTL
   is short and the link isn't cacheable, so hand each URL straight to transcription
   or download as you get it.
3. **Pace the fan-out — fetch one at a time with a beat between them.** Heavy reads
   throttle at ~5/min, and Binotel signals it as `code 106` inside a **200 body**
   (HTTP 429 is only a fallback some deployments don't emit). On either, back off and
   surface it — never blind-retry, or you burn the whole batch and the 15-min links
   expire mid-run.

## "How many did we miss today vs pick up?"

There is no answer-rate or missed-count endpoint — you fetch the day and bucket it
yourself into the sentence the manager wants.

1. `binotel_list_calls_per_day(day_in_timestamp=1782907200)` — any timestamp *inside*
   today resolves to the cabinet-local Kyiv day; pass a mid-day value (2026-07-01
   12:00 UTC) so a UTC-midnight instant can't slip onto the wrong local day. `None`
   also means today.
2. Bucket by the raw `callType` — `"0"` = incoming, `"1"` = outgoing (no `direction`
   field on the row; map it). Within incoming, picked-up = `disposition == "ANSWER"`
   & `billsec > 0`; missed = `{NOANSWER, BUSY, CANCEL, CONGESTION, CHANUNAVAIL}`.
   Now you can say it: **"picked up 37, missed 9."**
3. 📊 Same cap guard as above — if `extra.truncated` is set, that "9" is a floor, not
   the truth. Split today into hour windows via `binotel_list_calls_for_period`
   (`direction="incoming"` then `"outgoing"` — `all` is a *different* endpoint capped
   at 24h and is not the tool for a per-direction split) and sum. If the user wants
   *only* lost calls with nothing to compute, missed-today lives in the `calls`
   category, not here.

## "What's landed since my last sync?"

A running catch-up loop where each poll's high-water mark feeds the next — no row
delivered twice, nothing missed at the boundary.

1. First poll, "since 9am Kyiv this morning" → 2026-07-01 09:00 Kyiv = `1782885600`.
   `binotel_list_calls_since` has **no `all`**, so make two calls:
   `binotel_list_calls_since(timestamp=1782885600, direction="incoming")` then again
   with `direction="outgoing"`.
2. Read the returned rows' `startTime` values. Say the newest incoming row is
   `startTime: 1782889200` — advance your watermark to `max(startTime) + 1 =
   1782889201` so the boundary call isn't re-delivered next poll.
3. Next poll starts at `1782889201`; repeat. Each poll still returns ≤ 50 rows — if
   one saturates (`truncated`), shorten the interval so no burst is lost between
   checks.

## Boundaries you'll hit

- **"Find every call from +380671112233 last week"** — there is no by-phone history
  tool in stats. Pull the period or day list and filter rows on `externalNumber`
  client-side. That field comes back **raw** (`+380…`, `380…`, or `0…`), so normalize
  both sides before matching or you'll miss real hits.
- **Marketing-source calls this month** → `binotel_list_calltracking_for_period` over
  the window (CallTracking = Binotel's ad-attribution rows; same 50-row cap).
- **Who was on the other end** — a row's `customerData.id` is Binotel's *internal*
  customer ID; resolve full records via the `customers` category. Binotel stores
  phone + its own ID only — no native CRM-lead link, so join to your CRM by phone.

## Reference — dispositions (client-side, provider-fixed; match exactly)

`callType`: `"0"` = incoming, `"1"` = outgoing.

| `disposition` | Meaning | Recording? |
|---|---|---|
| `ANSWER` | Live conversation — a human talked | yes |
| `VM` / `VM-SUCCESS` | Voicemail deposit / left successfully | yes |
| `SUCCESS` | Generic success | yes |
| `NOANSWER` | Rang, not answered | — |
| `BUSY` / `CANCEL` | Busy / caller hung up before answer | — |
| `CONGESTION` / `CHANUNAVAIL` | Unreachable / channel unavailable | — |
| `TRANSFER` / `CALLING` / `ONLINE` | Transferred / dialing / in progress | — |

Recording-eligible = `{ANSWER, VM, VM-SUCCESS, SUCCESS}`; live conversation ("talked
to someone") = `ANSWER` + `billsec > 0` only; missed = `{NOANSWER, BUSY, CANCEL,
CONGESTION, CHANUNAVAIL}`.
