# Altegio appointments (records) — workflows

Altegio calls an appointment a **record**. Jump to the matching task. Steps
marked **✍** write to the schedule — confirm vague requests before firing one.

## Model note — read before acting

- **No availability check exists.** `altegio_create_appointment` posts straight
  to the schedule with no free-slot or conflict pre-check, so it silently
  double-books an occupied master. Always run the double-booking guard (below)
  before booking.
- **Resolve IDs from their own categories — never scrape the records list.**
  Get a master via `altegio_list_staff` (`staff`) and services via
  `altegio_list_services` (`services`, filterable by `staff_id` for only what
  that master performs). Names/titles in `altegio_list_appointments` are
  free-text and miss unbooked services.
- **Update edits only staff/datetime/seance_length/comment/attendance** — not
  services or the client. To change booked services, delete and recreate, then
  confirm the new record with `altegio_get_appointment` **before** deleting the
  original.
- **`seance_length` is SECONDS** (3600 = 1h); `30` books 30 seconds. Omit it to
  auto-compute from the services — the safe default. The list `summary` default
  already carries staff/services/client; use `include="full"` only beyond that.

## Task: book a client for a service with a master at a time

1. `altegio_list_staff(...)` → `staff_id`; `altegio_list_services(staff_id=
   <staff_id>)` → `service_ids`.
2. Avoid a duplicate client: `altegio_list_clients(query="+15551234567")`
   (`clients` category). Create sends an inline client `{name, phone}` and does
   NOT return or confirm a client_id, so you cannot tell afterward whether it
   reused an existing client — search first.
3. **Double-booking guard:** `altegio_list_appointments(staff_id=<staff_id>,
   start_date="2026-06-20", end_date="2026-06-20")` → for each row, add its
   `seance_length` (SECONDS) to its `datetime`; if any span overlaps the
   requested time, stop and tell the user instead of booking.
4. **✍** `altegio_create_appointment(staff_id=<staff_id>,
   service_ids=<service_ids>, datetime="2026-06-20 15:30:00",
   client_name="Jane Doe", client_phone="+15551234567", send_sms=True)`.

## Task: reschedule, mark no-show/arrival, delete

- Reschedule: run the double-booking guard for the new master/day, then **✍**
  `altegio_update_appointment(record_id=88, datetime="2026-06-20 17:00:00",
  staff_id=42)`.
- No-show / client-side cancellation — the default for both: **✍**
  `altegio_update_appointment(record_id=88, attendance=-1)`. It preserves
  history/payroll and fires the no-show notification, where a hard delete wipes
  all of that. Do NOT leave `attendance=0` ("waiting") to resolve a
  cancellation — it reads as a live pending slot. There is no fee field and no
  cancel-without-no-show status; fees are a `finances` concern.
- Arrived / confirmed: `attendance=1` / `attendance=2`.
- Delete only when the record must be gone entirely (booked by mistake): **✍**
  `altegio_delete_appointment(record_id=88, confirm=True)`.

## Gotchas

- **403 "No rights to manage location"** is permanent, not transient: the token
  lacks the records scope and must be re-minted with appointment rights.

## Reference — attendance enum (system-defined integers)

| Value | Meaning |
|---|---|
| `-1` | Did not arrive (no-show) |
| `0` | Waiting / pending — default of a new booking |
| `1` | Arrived |
| `2` | Confirmed |
