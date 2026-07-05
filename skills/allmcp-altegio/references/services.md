# Altegio services — workflows

The read-only service catalog: it turns "what does the salon offer / how long /
how much" into the IDs a booking needs. Jump to the matching task. ✍ marks a
write step in another category — confirm with the user before firing it.

## Model note — read before acting

- **Read-only catalog, no availability check.** Only `altegio_list_services` and
  `altegio_get_service` exist — adding/renaming a service or changing a price is a
  salon-admin back-office task, and free-slot lookup lives elsewhere; don't
  promise either here.
- **Durations are SECONDS.** `seance_length` and `duration` are in seconds —
  ÷60 for minutes, ÷3600 for hours (`1800` = 30 min, `3600` = 1 h). This is the
  category's top arithmetic trap; never read the number as minutes.
- **Price is a range.** Each service's `price_min`/`price_max` can differ, and
  masters override within that range — quote the per-master figure from the
  `staff` array (full payload) when you have it, not just the service-level range.
- **`include="summary"` (the list default) is lossy** — it drops the `staff`
  array, `sex`, and `prepaid`. For per-master price/duration, gender rules, or
  prepayment, use `include="full"` on list, or `altegio_get_service` (full by
  default).
- **The catalog is per-location** — read against the specific `company_id` you
  will book against; one location's menu does not apply everywhere.

## Task: browse the menu / quote a service's duration & price

1. `altegio_list_services()` (summary) → each row's `title`, `seance_length`
   (÷60 for minutes), `price_min`/`price_max`, `active`.
2. When the goal is to book or schedule, drop rows where `active` is falsy —
   a falsy `active` means hidden / not bookable online, so quoting it misleads.
3. `altegio_get_service(service_id=<id>)` to disambiguate when titles collide.

## Task: resolve service IDs to book (optionally for a named master)

`altegio_list_services` needs no `staff_id` — resolve the service and the master
independently; neither blocks the other.

1. (For a named master) `altegio_list_staff` (staff category) → `staff_id`, then
   `altegio_list_services(staff_id=<staff_id>)` for their services. That returns
   the service rows, not the master's overrides — read those from the `staff`
   array on `include="full"`.
2. Pick the service → `id` (and `seance_length` only if overriding).
3. Hand off to `altegio_create_appointment(service_ids=[<id>...])` (appointments
   category) — ✍ a write; confirm the service, master, and time with the user
   before firing. Omit `seance_length` there to auto-compute; pass it (seconds)
   to override.

## Cross-category & gotchas

- To filter one category, `altegio_list_services(category_id=<id>)` — but **no
  category-listing tool exists**; get the numeric `category_id` from an existing
  service's `category_id` field or the admin UI.
- Booking, staff-name→`staff_id`, client lookup/dedupe, and multi-location
  `company_id` live in the `appointments`, `staff`, `clients`, and `companies`
  categories — a "book X with Y" request is a multi-category chain.
- `altegio_get_service` on a missing ID raises a `ToolError`, not an empty item.
