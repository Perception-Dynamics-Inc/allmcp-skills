# Binotel customers — workflows

Jump to the job matching the user's request. This is Binotel's phone-keyed
mini-CRM, and two laws govern every recipe here because names and phone numbers
are unique cabinet-wide:

- **Find before you create.** A stored number lives in normalized `380…` form, so
  a raw `0…` lookup can whiff on a customer who already exists. A false "not found"
  is exactly what spawns a duplicate — which then slams into the uniqueness wall.
  So you search both forms before ever writing.
- **Confirm the write stuck.** Creates return an ambiguous ID and can raise on a
  record that actually landed; reassignments can silently no-op. So every step
  marked **✍** ends by re-reading the real value. Never report "done" off the
  write's return alone.

## Identify the inbound caller so the manager greets them by name

The caller on `+380671112233` rings in; hand the manager the existing account —
don't let the CRM spawn a second ghost record for the same person.

1. `binotel_search_customers(subject="0671112233")` — the raw local form the
   caller ID shows. Empty? Retry the stored form:
   `binotel_search_customers(subject="380671112233")`. Search sends `subject`
   stripped but never phone-normalized, so only when **both** forms come back
   empty is this caller genuinely new — that two-form pass is the one thing
   standing between you and a duplicate.
2. The match is Olena Kravets, `id` `6611`. The summary row already flattens her
   manager to top-level `assignedToEmployeeID` — route the call without an
   `include="full"` re-fetch. (Holding a `customerData.id` from a call log instead?
   `binotel_get_customers(customer_ids=[6611])` — it returns a list even for one ID.)

## Land this morning's lead as one customer, owned by the right manager

Petro Marchenko phoned in from `+380509998877` and gets booked into the mini-CRM
exactly once, owned by manager Andriy, so his next call routes straight to Andriy.

1. `binotel_list_employees()` (employees category, auto-unlocks) → find Andriy and
   read his `employeeID` `"58"`. Feed that literal string into
   `assigned_to_employee_id` — not his extension `0901`, not an int. Binotel keeps
   these as strings.
2. Prove Petro is new: search both phone forms — `subject="0509998877"` then
   `subject="380509998877"` (recipe above). A skipped search here is the classic
   duplicate.
3. **✍** `binotel_create_customer(name="Petro Marchenko", numbers=["0509998877"],
   email="petro@acme.ua", assigned_to_employee_id="58")`. The number is normalized
   to `380509998877` on the way in; up to 10 numbers per write.
4. The create came back with a raise or an ID whose field is ambiguous
   (`customerID` vs `id`) — you do not yet know the lead exists. Do not blind
   retry: a retry hits the phone-uniqueness wall and now an error masks a record
   that may already be live. Instead re-search `380509998877`: exactly one row
   owned by `"58"` → created, report it; zero rows → the create truly failed, now
   safe to retry.

Phone hygiene that silently corrupts the record if you skip it: give foreign
numbers in international form (`+48…`), never local — normalization is Ukraine-only.
A whitespace-only entry in a multi-number list is dropped without a word; an
all-empty list or a sub-7-digit number raises `ToolError`.

## Hand the manager the correct debtor list for collection calls

The manager starts collection calls off the real debtor list — pulled by the
actual `labelID`, never a guessed one.

1. `binotel_list_customer_labels()` → each row carries an `id` and a `name`, e.g.
   a debtor tag `{"id": "3", "name": "Боржник"}` beside `{"id": "7", "name": "VIP"}`.
   Debtor tags read as `Боржник` (UA) or `Должник` (RU); match the `name`
   case-insensitively. On genuine ambiguity, ask rather than guess a `labelID`.
2. `binotel_list_customers_by_label(label_id=3)` — the id reads as the string
   `"3"` in the dictionary, but this param takes an `int`, so pass `3`.

When the manager then asks to actually tag or untag someone: you can't from here.
No tool writes labels — `create`/`update` don't accept them, and faking one in
`description` is not a tag. Send them to the Binotel cabinet UI for that.

## Reassign Olena's account to Dmytro and confirm it stuck

Olena is leaving; her account becomes Dmytro's, confirmed live, so no one keeps
calling her about it.

1. Resolve the account's `customer_id` first — it's the `6611` you found for the
   caller, or a call log's `customerData.id`. Get Dmytro's `employeeID` `"61"` from
   `binotel_list_employees`.
2. **✍** `binotel_update_customer(customer_id=6611, assigned_to_employee_id="61")`.
   Only non-None fields are sent, so this touches nothing else.
3. **✍** Read it back: `binotel_get_customers(customer_ids=[6611])` and confirm
   `assignedToEmployeeID` now reads `"61"`. This is non-negotiable — the connector
   writes `assignedToEmployeeID` while the native docs show `assignedToEmployeeNumber`,
   so a reassignment can silently no-op. The read-back is the only thing between
   "reassigned" and a lie.

You can hand the account to Dmytro, but you cannot make it ownerless — an empty
employee id is stripped, never sent. There is no un-assign here; the cabinet UI is
the only place for that.

If instead you're changing phones: sending `numbers` **replaces the whole list**.
The account holds `["380671112233", "380442334455"]` and the user wants to add a
mobile — passing `numbers=["380637775544"]` alone silently deletes the other two.
Pass all three: `numbers=["380671112233", "380442334455", "380637775544"]`. Then
read back the record and confirm every number survived.

## Safely remove the three onboarding test records without touching anything real

1. `binotel_get_customers(customer_ids=[9001, 9002, 9003])` and show the user
   exactly which names and numbers will die, then get explicit approval — delete
   is irreversible, so no silent destructive writes.
2. **✍** `binotel_delete_customers(customer_ids=[9001, 9002, 9003], confirm=True)`.
   The `confirm=True` gate is mandatory; ≤50 IDs per call.

## Pulling more than 50, without tripping the rate limit

Every list caps at 50 rows and cannot page — `next_cursor` is always null. Past 50,
narrow with `binotel_search_customers` or `binotel_list_customers_by_label`, and
check `extra.truncated` / `total_available` before trusting a count. These `list`/
`search` calls are resource-heavy: pace them so a burst mid-collection-run doesn't
trip Binotel code `106` (HTTP 429 is the fallback), surfaced as a `ToolError` —
back off, don't blind-retry.

## No external-CRM linkage

There's no link to a Bitrix24/AmoCRM/SalesDrive record — Binotel keeps only its
own `customerID` + phone `numbers`. Match by phone in that other connector.
