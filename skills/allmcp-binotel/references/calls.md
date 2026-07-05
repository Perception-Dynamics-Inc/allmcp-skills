# Binotel calls — get the job done

People come here for four finished results: **one conversation played back** to settle a QA dispute; **a caller's full prior history** in front of the rep before they finish saying hello; **who is live on the phones right now**; and **today's missed numbers ready to ring back** before the callers give up. Where they want audio, the job isn't done until a *fresh* recording link is in the transcription pipeline.

Two things silently wreck all four, and every recipe below is built to beat them: a truncated slice handed over as the whole history, and a recording link that rotted before it was used. `calls` does point lookups only — anything with a date, week, or per-rep window is a `stats` job (below); reconstructing a window by paging history here is impossible, because nothing here pages.

## The manager has this week's answered calls for one rep, each with a live transcription link

The manager says "pull Oksana's calls this week and send me the recordings." A rep name and a week window is a `stats` job first — `calls` has no employee or date filter. Resolve the person, list the week in `stats`, then win the recording race here.

1. `binotel_list_employees()` (**`employees`** category) → find "Oksana Koval" → her `extensionNumbers` is `["0901"]`. Keep the leading zero — `"0901"` and `"901"` are different extensions.
2. `binotel_list_calls_for_employee(internal_number="0901", start_time=1719176400, stop_time=1719705600)` (**`stats`**). Those integers are Mon 00:00 Kyiv through the following Sunday — a 6-day span. The window must be **≤ 7 days (604800s)**; hand it a wider one and the tool rejects it outright with a ToolError ("window of Ns exceeds the 604800s (7-day) cap") — it never quietly clamps, so you'll know instantly you overreached. For a real month, slice it into 7-day chunks and stitch. Rows carry `disposition`, `callType`, `billsec`, `waitsec`, `startTime`, `externalNumber`, `generalCallID` — everything to bucket and count with no extra call.
3. Keep `disposition == "ANSWER"` only. The manager wants calls where the rep *spoke with a human*. A `VM-SUCCESS` row has a recording, so it's tempting — but it's the rep talking to an answering machine. Slip it in and the "answered calls" count is padded with voicemails the manager reviews expecting a conversation that never happened.
4. That leaves, say, 8 answered rows. **Fetch the recordings last, right before handoff** — the signed MP3 URL lives ~15 minutes, then it's a dead link. For each kept row: `binotel_get_recording_url(general_call_id="2255713", disposition="ANSWER")`. Pass the disposition you already hold; omit it and a stray `NOANSWER` sails across the network only to fail on a missing URL, spending a rate-limit slot for nothing.
5. 8 fetches means pacing. Heavy methods throttle at **~5/min** (Binotel code 106, or HTTP 429 — both reach you as a ToolError). Fire ~5, pause, finish the rest. Blast all 8 back-to-back and Binotel trips around the 6th — you deliver 5 links and a mid-batch error. If an answered call returns no URL, surface it as "1 recording unavailable" and move on; don't loop.

→ Manager receives: 8 answered calls, 8 live transcription links (or "7 of 8, 1 unavailable"), 0 voicemails miscounted, 0 rows silently dropped.

## A manager wants one specific conversation pulled up to settle a dispute

QA flagged a shouting match on call `2255713` from yesterday's export; the manager wants to hear it and read the durations.

1. `binotel_get_calls(general_call_ids=["2255713"])` → the detail row. Pass the ID in a list even for one — the API expects an array. **Zero matches raises a ToolError**, so a bad ID is a hard error, not an empty "call not found" you paper over.
2. Read the row you already hold — no second call. `billsec=214` is billed talk time, `waitsec=18` is ring/wait; total on the line is their **sum**, 232 seconds. `callType` is raw: `"0"` = incoming, `"1"` = outgoing (there is no `direction` field). `startTime` is Unix seconds **UTC** — convert for a Kyiv manager thinking in local time.
3. `disposition` is `"ANSWER"`, so a recording exists. `binotel_get_recording_url(general_call_id="2255713", disposition="ANSWER")` → forward the URL immediately.

→ Manager receives: the 3½-minute inbound call, its exact durations, and a live link to the audio.

## A customer is on the line — the rep has their full prior history before hello

The rep's phone rings from `+380671112233`.

1. Strip to E.164-without-plus and search: `binotel_search_calls_by_phone(external_numbers=["380671112233"])`. The `externalNumber` on returned rows comes back **raw** — it may read `0671112233` or `+380671112233`, so match on it loosely, never by exact string equality against your query.
2. If you already resolved this caller to a Binotel customer (e.g. "ТОВ Ромашка" → `customerID` `88214`, resolved in the **`customers`** category via `binotel_search_customers`), search by that instead: `binotel_search_calls_by_customer(customer_ids=[88214])`. That's a Binotel-table integer, not a Bitrix24/CRM contact ID — never feed a CRM ID here.
3. **A busy number can have hundreds of prior calls; this tool returns at most 50 rows and cannot page.** Before you tell the rep "here's their whole history," read `extra.truncated` — if it's set, you're holding a slice. Say "showing the 50 most recent of ~180" and pull the full span from a `stats` window instead. Quoting a truncated slice as complete is how a rep rings back sure they've seen everything.
4. Empty `items` is **not** proof of no history — a phone search can come back empty on a technical miss. Report "no prior calls found," never "this is a new number," unless the customer search agrees.

→ Rep receives: the caller's real prior calls in hand, flagged clearly if it's only the most-recent 50.

## The team has today's missed numbers, sorted to ring back the ones going cold

"Who did we miss today?"

1. `binotel_list_lost_calls_today()` → today's missed calls since **cabinet-local midnight**. No date param exists — there is no "missed on a date" here. For any other day, list via `stats` and filter to the missed set `{NOANSWER, BUSY, CANCEL, CONGESTION, CHANUNAVAIL}`.
2. Each row's `externalNumber` is a number to call back. Sort by `isNewCall`: a `true` row is a first-time caller who may never call again — ring those first; a `false` row is a known customer who'll likely try again.
3. If the list comes back empty, confirm it's genuinely a quiet day before reporting "nobody missed" — treat an unexpected `[]` with the same caution as the phone search.

**Fanning missed numbers into one `search_calls_by_phone` does not multiply the ceiling** — pass 10 numbers and the *output* still caps at 50 total rows, so some numbers' calls silently vanish from the result. Batching inputs never raises the cap; look each number up in its own right when completeness matters.

→ Team receives: today's missed numbers, new callers first, ready to dial.

## Who is on the phones right now

`binotel_list_online_calls()` → a live snapshot of in-progress calls, no params. This is a read-only wallboard. To actually place, transfer, or hang up a live call, that's the **`outbound`** category — nothing here mutates a call.

## The disposition filter — what lands in the report

Rows return `disposition` and `callType` **raw and un-humanized**. A wrong client-side guess like `ANSWERED` or `NO-ANSWER` matches zero rows and never errors — use the exact tokens below.

| `disposition` | Meaning | Recording? |
|---|---|---|
| `ANSWER` | Answered — live human conversation | Yes |
| `VM` | Went to voicemail | Yes |
| `VM-SUCCESS` | Voicemail recorded (no live talk) | Yes |
| `SUCCESS` | Generic success outcome | Yes |
| `NOANSWER` | Rang, not answered (one token, **not** `ANSWERED`) | No |
| `BUSY` / `CANCEL` / `CONGESTION` / `CHANUNAVAIL` | Not connected | No |

- **Live human conversation = `ANSWER` only.** `SUCCESS` is a generic success, not proof someone talked — never promote it into the "spoke with a human" bucket.
- **Recording exists for `{ANSWER, VM, VM-SUCCESS, SUCCESS}`.** Any other disposition has no audio; `get_recording_url` rejects it.
- **Missed = `{NOANSWER, BUSY, CANCEL, CONGESTION, CHANUNAVAIL}`.** `SMS-*`, `TRANSFER`, `ONLINE`, `CALLING`, `FAILED` also exist but aren't voice outcomes you filter on here.

`include="full"` only un-drops noise fields — it never adds a recording link; audio is *only* via `binotel_get_recording_url`. A row carries Binotel's own `customerData {name, id}` and a phone, never a CRM lead/deal ID — match to a CRM lead by phone against a separate connector (Bitrix24 / AmoCRM / SalesDrive).
