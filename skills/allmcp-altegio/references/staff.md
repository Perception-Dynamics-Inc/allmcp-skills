# Altegio staff (team members) — workflows

Jump to the task matching the user's request. A "staff member" is the service
provider — barber, stylist, master. Steps marked **✍** change data; if the
request was vague, confirm specifics before firing one.

## Model note — read before acting

- **Resolve the salon first in multi-location accounts.** Every tool defaults to
  the connected salon's `company_id`, and `altegio_list_staff`/`altegio_get_staff`
  strip `company_id` from output, so you cannot tell which salon a row is in. If
  the account may have more than one location, call `altegio_list_companies`
  (`companies` category) first and pass the chosen `company_id` on every staff
  call; otherwise the task silently scopes to the default salon and the person
  you want may not be in it.
- **No name search.** `altegio_list_staff` has no name/search filter — only
  `page`/`count` paging. Find by name by paging (`count=200`) and matching
  `name`/`specialization` on the summary rows client-side, before issuing
  per-candidate `altegio_get_staff` calls. The account allows ~200 req/min and
  5/sec, so don't fan out `altegio_get_staff` in parallel.
- **Capability boundaries — don't promise these.** No schedule tool (no shifts,
  days off, rota). No staff↔service linking (can't set which services a master
  performs or a per-master price — fold the skills into free-text
  `specialization`). No positions catalog (`position_id` can't be resolved here —
  leave it unset, never guess). `rating` is read-only. Phone is settable only at
  create.
- **`hidden` and `fired` are independent.** Setting one does not imply the other:
  to remove someone from the booking page AND keep history, set both.

**Finding a staff `id`** (every recipe below starts here): resolve `company_id`
if multi-location, then `altegio_list_staff(count=200, company_id=<id>)` and
match `name` in `items`. An empty result on the default salon is not proof of
absence — say you may have the wrong location. `altegio_get_staff(staff_id=<id>)`
only when you need the full profile.

## Task: dismiss an employee who left, or hide one temporarily

Never delete — `altegio_delete_staff` is a hard, irreversible erase that wipes
the master's stats, payroll, and booking history. Dismiss instead.

1. Find the staff `id`.
2. **✍** Dismiss permanently (keeps all stats):
   `altegio_update_staff(staff_id=<id>, fired=1, hidden=1)` — set `hidden=1` too,
   since `fired` alone is not guaranteed to pull them off the booking page. For a
   temporary holiday/sabbatical hide that keeps them on the active roster, use
   `hidden=1` only (admins can still book them manually in the calendar).
3. `altegio_get_staff(staff_id=<id>)` → confirm `fired`/`hidden` read back as you
   set them before reporting done. Reverse with `fired=0, hidden=0`.

## Task: hire / add a new staff member

**✍** `altegio_create_staff(name="Yana Petrova", specialization="Barber — fades
& beard trims", phone_number="+15551234567", company_id=<resolve if
multi-location>)` → new `id`. Add `user_email="yana@salon.com",
is_user_invite=True` to invite them as a system user in the same call. Leave
`position_id` unset unless the user gives the exact numeric id.

## Task: edit a profile / change a phone / reorder the roster

1. Find the staff `id`.
2. **✍** `altegio_update_staff(staff_id=<id>, name=..., specialization=...,
   information="Public bio shown to clients", weight=100)` — only fields you pass
   are sent. Passing `""` to blank a field is rejected, not honored.

**Phone cannot be edited** — there is no phone field on `altegio_update_staff`;
it is settable only at `altegio_create_staff`. Read the current phone via
`altegio_get_staff(include="full")` (the `summary` rows drop contact fields), and
frame the gap as a missing update path here, not as Altegio being unable to.

A 403 on a write means the connected user token lacks that access right (reads
often work where writes don't), not a bad `staff_id`.

## `staff_id` feeds other categories

- `appointments`: `staff_id` on `altegio_create_appointment` /
  `altegio_list_appointments`.
- `finances`: `staff_id` on `altegio_list_transactions` attributes income/
  expense to a staff member.
- `services`: `altegio_list_services(staff_id=…)` returns what that master
  performs (not on the staff object).
