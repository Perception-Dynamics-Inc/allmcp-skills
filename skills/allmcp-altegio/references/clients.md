# Altegio clients — workflows

Jump to the task matching the request. **✍** steps create/change/delete a client;
confirm specifics before firing one on a vague request. To target a non-default
location, discover its `company_id` via `altegio_list_companies` (`companies`
category).

## Model note — read before acting

- **These tools edit the client card only.** No tool sets a client's
  category/loyalty tier (Regular/Loyal/VIP) or adjusts balance/deposit/loyalty
  card — those are read-only stats here. No merge tool. Visits, payments, and
  no-show status live in other categories (see Gotchas). Don't promise an edit
  this category can't make.
- **Bookings auto-create clients.** Creating an appointment with a new
  name+phone spawns a client record, so after a booking the client already
  exists. `altegio_create_client` is for manual/walk-in entry only — always
  search by phone first or you make duplicates.
- **`include`/`efficiency_on` decide what comes back.** The summary shape OMITS
  `comment` — read an existing `comment` with `include="full"`, never
  `"summary"`. Visit stats are always present in either shape.
  `efficiency_on=False` returns the fully raw record (all stripped noise back) —
  it's the way to surface `custom_fields`/`documents` when a salon stores data
  there; leave it default otherwise.

## Task: find a client by name or phone

1. `altegio_list_clients(query="+380671234567")` — matches name/phone/email →
   scan `items` → `id`. The summary already carries visit stats (`visits_count`,
   `last_visit_date`, `sold_amount`, `paid`, `balance`), so answer "when/how
   much" straight from here — no `get_client` round-trip for stats.
2. Phone matching may be exact-string: `+380…` and `380…` could be stored
   separately. If a phone search is empty, retry digits-only (drop the `+`)
   before concluding the client is new.

## Task: add a walk-in without making a duplicate

1. Search first per the task above, both `+380…` and digits-only forms. Any hit
   → reuse that `id`, stop.
2. Both empty only: **✍** `altegio_create_client(name="Olena",
   surname="Petrenko", phone="+380501112233")` → `id`.
3. If create raises "did not return a client ID … phone may already belong", the
   phone is taken — re-search (both formats) and reuse it; never retry the create.

## Task: update contact info or a note

Partial write — only passed fields are sent, and blank/`""` is rejected (you
cannot clear a field through these tools); omit fields to leave them untouched.

1. **✍** Fix a phone/email: `altegio_update_client(client_id=42,
   phone="+380501119999")`.
2. **Appending to a `comment` overwrites it**, so round-trip:
   `altegio_get_client(client_id=42, include="full")` → read `item.comment` →
   **✍** `altegio_update_client(client_id=42, comment="<old> · <new>")` → then
   `altegio_get_client(client_id=42, include="full")` again to confirm the merged
   note saved. Skipping `include="full"` silently drops the existing note (summary
   has no `comment`).

## Task: clean up a duplicate

No merge exists; the only remedy is to delete the empty duplicate — a hard,
irreversible delete (drops profile + history, no undo, no read-back). Search to
identify both, compare `visits_count`/`sold_amount` to find the empty one,
confirm the `id` with the user, then **✍**
`altegio_delete_client(client_id=<dup>, confirm=True)`.

## Gotchas

- **No-show is on the appointment, not the client:** set `attendance=-1` via
  `altegio_update_appointment` (`appointments`) — never delete the appointment.
- **Cross categories with the `id`** for a client's visits
  (`altegio_list_appointments(client_id=…)`, `appointments`) or payments
  (`altegio_list_transactions(client_id=…)`, `finances`).
- **Booking for a found client:** resolve staff (`altegio_list_staff`, `staff`)
  and services (`altegio_list_services`, `services`), then
  `altegio_create_appointment` (`appointments`) — it posts to the calendar with
  NO availability check, so first list the staff member's records for that day
  (`altegio_list_appointments(staff_id=…, start_date=…, end_date=…)`) to avoid
  double-booking.
- **A 403** means the token lacks "clients" rights, not a wrong ID — re-mint the
  token with client access; other IDs won't help.

## Reference

`summary` keep-list: `id, name, surname, phone, email, discount, visits_count,
first_visit_date, last_visit_date, sold_amount, paid, balance`. `comment` and
`custom_fields` are NOT here — use `include="full"` for `comment`,
`efficiency_on=False` for `custom_fields`. Client categories (Regular/Loyal/VIP)
are read-only and portal-configurable — read an existing client to learn a
salon's actual tiers.
