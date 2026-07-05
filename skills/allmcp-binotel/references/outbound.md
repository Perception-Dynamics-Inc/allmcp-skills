# Binotel outbound — making the call actually happen

This is where "please call them" becomes "the call happened, and here's what
actually came of it." Every dialing tool answers only `{success, message}` — a
`success=True` proves Binotel *accepted the dial request*, nothing more: no phone
rang, no human picked up, no recording played, and no call ID came back. The job
isn't finished when the dial returns; it's finished when you've pulled the call
row and read the real disposition. Report a dial as **dialing / accepted** — never
*reached / answered / delivered* — until that row confirms it.

Two rules hold across every recipe below:

- **Resolve the unguessable input before you dial**, from its sibling category:
  the extension and the `voiceFileID` from `employees`, the `generalCallID` from
  `calls`. Call those tools directly — dispatch auto-unlocks them. The one input
  with **no lookup anywhere** is `ivr_name`, a portal-only scenario name: ask the
  user, never guess.
- **Ukrainian numbers dial fine in any form** (`0671112233`, `+380671112233`,
  `380671112233` all normalize). A **foreign** number must be full international
  `+<country>...` form — a foreign *local* number is silently mishandled and the
  call goes nowhere you intended.

**✍** marks a step that dials or ends a real call. The `confirm=True` gate is
carried by the tools themselves, so the judgment worth words is *which number
you're about to dial* — see the bridge and transfer recipes, where a wrong target
rings a stranger.

## Get Marina on the phone with the lead — her desk rings, then the lead

The manager clicks "call" on a CRM lead; her own extension rings first, and by the
time she lifts the handset the lead's mobile is already dialing.

1. `binotel_list_employees()` → find Marina's row → `extensionNumbers: ["0901"]`.
   That `"0901"` is a **string with its leading zero intact** — it is her SIP
   extension, not a number to do math on.
2. **✍** `binotel_initiate_internal_to_external(internal_number="0901",
   external_number="+380671112233", confirm=True)`. Omit `pbx_number` — for this
   endpoint Binotel falls back to the cabinet's default trunk as caller ID.
3. Read it back (below) over Marina's extension — her phone physically rang, but
   the *lead's* answer state only shows in the row.

## Put the customer and the supplier on one line

No employee sits in this call — Binotel bridges two outside numbers directly, so
there is no desk phone ringing to tell anyone it worked. Both legs dial on a
trunk you must name.

1. **✍** `binotel_initiate_external_to_external(external_number_1="+380671112233",
   external_number_2="+380509998877", pbx_number="0442334455", confirm=True)`.
2. `pbx_number` is **required here — Binotel rejects this endpoint outright without
   a trunk**, no default to fall back on. `0442334455` is a cabinet trunk (the
   outbound line customers see as caller ID), addressed phone-number-style; if you
   don't know it, ask — don't invent one, and if a phone-form value errors, the
   cabinet may use a trunk ID/name instead, so ask for the exact value.
3. Both legs are strangers' phones — the highest-exposure dial in the module.
   Confirm **both** numbers with the user before firing, then read both legs back
   (below) before you call it connected.

## Hand a caller you dialed into the support queue

You dial a customer, and the moment they answer they drop straight into the
inbound IVR/queue exactly as if they'd called in — a warm handoff with no rep
holding the line.

1. **✍** `binotel_initiate_external_to_incoming_call(external_number="+380671112233",
   pbx_number="0442334455", confirm=True)`. Same **required trunk** as the bridge
   above — no trunk, no call.
2. Leave `phone_number_for_incoming_call` unset unless the user names a specific
   inbound route; unset means the cabinet's configured incoming flow catches them.
3. Read it back (below) — the row shows whether they answered into the queue or
   dropped before pickup.

## Prove every client on the list actually heard the payment reminder

The manager wants 500 delinquent clients auto-dialed with the recorded reminder —
and wants to know *which ones actually got it*. There is **no bulk dialer**: no
list param, no scheduler, no retry loop. The finished job is 500 dials placed one
at a time, and 500 dispositions read back. "The blast went out" is not the
deliverable; "437 played, 63 no-answer, here's the breakdown" is.

1. `binotel_list_voice_files()` (`employees` category) → every row carries a
   `voiceFileID` (e.g. `"81542"`); a name/label field is *not* guaranteed, so match
   the user's recording on a name/label field only if the row carries one. If
   nothing matches the recording the user named — or the rows are bare IDs with no
   readable name — **stop and ask which `voiceFileID`**; do *not* dial a guessed
   file, and do *not* bounce this back as "I can't find it," because this lookup
   exists and is one call away. No tool creates or edits a voice file.
2. Per client, **✍** `binotel_call_with_announcement(external_number="+380631234567",
   voice_file_id="81542", confirm=True)`. Loop it yourself, one number at a time.
   Note there is no caller-ID/trunk param here — you cannot force which line shows.
3. Per client, read back (below). A `success=True` on the announcement means *only*
   that Binotel queued the dial — whether the recording played is entirely in the
   disposition. Report the batch as **accepted**, then as **delivered** only after
   the rows come back.

For an *interactive* confirmation call the client answers with a keypress, use
**✍** `binotel_call_with_ivr(external_number="+380631234567",
ivr_name="delivery-confirm", confirm=True)` instead. `ivr_name` is the ONE
input with no resolver — no tool lists scenario names — so you get it from the
user, never from a guess. (Optional `text` plays a TTS lead-in before the menu.)

## End the stuck call, or send the angry caller to someone who can help

A call is hung on Marina's line, or a caller needs the specialist — you act on the
*live* call, which the initiate tools never handed you an ID for.

1. `binotel_list_online_calls()` (`calls` category) → find the live call →
   `generalCallID: "18400000001234"`. The dial tools do not surface this; this
   read is the only place it comes from.
2. **Kill it:** **✍** `binotel_hangup_call(general_call_id="18400000001234")`.
   Hangup has **no `confirm`** — it only reduces exposure, so there's nothing to
   gate.
3. **Move it:** **✍** `binotel_transfer_call(general_call_id="18400000001234",
   external_number="+380442334455", confirm=True)`. This **dials a new external
   number** — get the transfer target wrong and the live caller lands on an innocent
   stranger, so confirm that exact number with the user first. Unlike the IDs above,
   the transfer target is a judgment call, not a lookup.

## Read the truth back — the last step of every recipe above

To turn "accepted" into what actually happened, pull the call row from a
`stats`/`calls` read tool, keyed by number or extension.

1. **By number** (a blast leg, a bridge, a transfer):
   `binotel_search_calls_by_phone(external_numbers=["380631234567"])` (`calls`).
   **By extension over a window** (Marina's calls this week):
   `binotel_list_calls_for_employee(internal_number="0901",
   start_time=1719792000, stop_time=1720310400)` (`stats`) — Unix seconds, UTC, a
   6-day window well inside the hard **≤ 7-day** cap; slice longer ranges into
   7-day chunks yourself.
2. Read each row's `disposition` — a closed enum, not free text:
   - **A live human talked** = `ANSWER`. The win for a click-to-call, a bridge, or
     a transfer — the point was a real conversation.
   - **Delivered** = `ANSWER`, `VM`, `VM-SUCCESS`, `SUCCESS` — a person *or* an
     answering machine that took the recording. For an announcement blast all four
     count as delivered; never report a `SUCCESS`/`VM-SUCCESS` leg as "didn't reach
     anyone."
   - **Not connected** = anything else (`NOANSWER`, `BUSY`, `CANCEL`, `CONGESTION`,
     `CHANUNAVAIL`, `FAILED`, …) — those need a retry, not a "done."
3. For the recording, feed a row whose disposition is one of
   `{ANSWER, VM, VM-SUCCESS, SUCCESS}` — **and only those; every other disposition
   has no audio** — into `binotel_get_recording_url(general_call_id=...)` (`calls`).
   The signed MP3 dies in ~15 minutes and isn't cacheable: hand it straight to
   transcription, don't stash it in a report you'll send later.

## Don't promise what this category can't do

- **No SMS.** Nothing here texts a customer — don't offer a text fallback.
- **No conference.** A bridge is exactly two legs; there's no 3-way.
- **No editing voice files or IVR scenarios.** You can *list* voice files, never
  create/record/edit one, and can't enumerate IVR scenario names at all — those
  live in the cabinet portal.
- **No reachability/DND pre-check.** Numbers are normalized and shipped; whether
  the callee consents or is reachable is on the caller.
