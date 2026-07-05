# Binotel employees — workflows

This directory has one real job: turn a name a manager says out loud into the
exact value the stats and outbound tools will accept — the extension that pulls
a week of calls, the `voiceFileID` that plays on an outbound dial, the
`employeeID` that flips a presence state. Get the resolution right and the
downstream category just works. Get it wrong and it fails silently, handing the
manager a confidently wrong answer. **✍** steps mutate real state.

## Hand the manager Марина's week of calls — resolved to the extension that pulls them

The manager asks for Marina's week so the recordings can go to transcription.
That call history lives in the `stats` category — your job here is to resolve her
to the one value that unlocks it, so the next tool returns her calls and no one
else's.

1. `binotel_list_employees(include="summary")` — the default `summary` shape
   already carries `extensionNumbers` and `presenceState`, so this one pull is
   enough. There's no server-side search or filter, so read the whole roster and
   scan `name` yourself for "Марина Ковальчук" → `employeeID` `"58"`,
   `extensionNumbers` `["0901"]`. If the cabinet runs past 50 people the tail is
   truncated with no page 2 (`extra.truncated` set, `next_cursor` always
   `None`) — if you don't find her, tell the manager the roster is partial, don't
   swear she isn't there.
2. She has one extension, so the handoff value is unambiguous:
   `extensionNumbers[0]` = the string `"0901"`. Pass it verbatim — the leading
   zero is a live SIP digit; if it gets coerced to `901` the stats lookup
   returns nothing and no one tells you why.
3. Feed that exact string as `internal_number` into the `stats` tool with a
   concrete ≤7-day window (Unix seconds, UTC). For this Mon–Sun Kyiv week
   (2024-07-01 00:00 → 2024-07-07 23:59 EEST):
   `binotel_list_calls_for_employee(internal_number="0901",
   start_time=1719781200, stop_time=1720385999)` returns exactly her week — each
   row's `generalCallID` is what transcription needs: hand it to
   `binotel_get_recording_url` (calls category) to mint the signed MP3 link.
   Recompute both stamps for the actual week the manager means; the window must
   stay ≤ 604800 seconds or stats rejects it.

The handoff key is the **extension**, never the `employeeID`. Hand `stats` the
`"58"` and it comes back empty or full of someone else's calls — and you'd
report a confidently wrong week. `"0901"` is Marina's calls; `"58"` is garbage.

If a row shows two lines — `extensionNumbers` `["0901", "0912"]` — decide, don't
guess: pull calls for each and union them if both are hers, or confirm which
line the manager means before reporting, since one extension is half her week.

## Take Олег off the dialer for his lunch break — and prove it landed

Oleg is stepping out for 30 minutes. The supervisor needs the auto-dialer to
stop routing calls to his line — and needs to *know* it actually stopped, not
just that the request was accepted.

1. `binotel_list_employees` → scan `name` for "Олег Сидоренко" → `employeeID`
   `"63"`. Resolve the specific Oleg first: this write has **no confirm gate**,
   it fires the instant you call it, so a vague "set the team to break" would
   flip whoever the model guessed with nothing to catch it.
2. **✍** `binotel_change_employee_presence_state(employee_id="63",
   presence_state="break in work")`. Pick `"break in work"`, not `"inactive"` —
   he's back in half an hour, so this pauses routing; `"inactive"` reads as
   off-for-the-day. The four states are `active`, `work in crm`, `break in work`,
   `inactive`, exact and lowercase; anything else is rejected before the HTTP
   call. (These four are authoritative from our connector, not a public doc — if
   the API rejects one, surface that rather than retrying blindly.)
3. **Read back — this is the moment the job is done.** `success=True` only means
   the POST didn't error; it does not mean the state changed. Re-run
   `binotel_list_employees` and confirm Oleg's row now reads
   `presenceState == "break in work"`. Only then is his line truly off the
   dialer — a `success=True` that never landed leaves calls hitting a phone
   nobody's answering, and you'd have told the supervisor it was handled.

Flipping the whole team at shift change is the same move, one write at a time
with a beat between them. Binotel throttles this directory as a "heavy" method
at ~5 calls/minute (~5s gap); burst it and you get a `ToolError` for rate limit.
Whether the presence write shares that bucket is unverified — so sequence
conservatively rather than firing writes in parallel, or the back half of your
team silently never gets flipped. Read back each row after.

## Resolving values for adjacent jobs

Same directory, same resolve-then-hand-off pattern:

- **Playing an announcement on an outbound call.** `binotel_list_voice_files`
  returns the `voiceFileID` for a recorded prompt — hand that ID to
  `binotel_call_with_announcement` (outbound, confirm-gated) so the campaign
  plays the right message. Uploading or renaming the prompt happens in the
  Binotel cabinet, not here.
- **Checking how incoming calls are routed.** `binotel_list_routes` reads the
  cabinet's ring scenarios. Both voice files and routes are pass-through, so
  `summary` and `full` return the same fields — reach for `full` only when you
  genuinely need a field you can't see. Routes and employee profiles are
  read-only through this connector; edits happen in the cabinet.
