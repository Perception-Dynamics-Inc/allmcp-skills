# Altegio finances — workflows

These tools record **manual finance ledger entries only**. Jump to the matching
task. Steps marked **✍** create, change, or delete ledger data — if the request
is vague, confirm first.

## Model note — read before acting

- **NOT a checkout, refund, or register operation.** `altegio_create_transaction`
  writes a manual bookkeeping entry — it does NOT close a visit, mark an
  appointment paid, print a fiscal receipt, refund a client, or touch inventory.
  A client paying for a service is a checkout on the salon's register (the KKM
  side), not a manual transaction. When asked to "refund the client", "mark this
  paid", or "undo their payment", say this connector cannot do it and route them
  to the salon's register flow. A negative manual entry is bookkeeping, not a
  refund — no "Refund"/«Возврат» article ships, so don't book one under an income
  article with a comment.
- **Two unguessable IDs gate every create — resolve them, never invent.**
  `account_id` from `altegio_list_accounts`; `expense_id` from
  `altegio_list_expense_categories`. A wrong/missing pair returns no ID and a
  ToolError.
- **The article type — not a comment or sign — sets income vs expense.** Each
  article is typed income or expense in salon settings, so pick the article whose
  type matches the direction. A minus sign or a "refund" comment can't turn an
  income article into an outflow. Default to a positive `amount`; go negative only
  if the salon's own article convention requires it (the sign rule is
  configurable, not universal).
- **List rows carry nested objects, not flat ID keys.** The default
  `include="summary"` row is `{id, date, amount, comment, account{id,title},
  expense{id,title}, client{id,name}, master{id,name}}` — already enough to sum
  money and group by master or client. There is no top-level `staff_id`/
  `master_id`/`client_id`: attribute by `master.id`/`master.name` and
  `client.id`/`client.name`. Use `include="full"` only for a raw field the
  summary drops, never just to get `amount`.
- **`staff_id` is an input filter** that maps to the API's `master_id`; you read
  it back as `master{id,name}`.

## Task: record a payout, expense, or income

"Pay Marina 5000 from the drawer" / "we spent 1200 on shampoo".

1. `altegio_list_accounts()` → pick the register → `account_id`.
2. `altegio_list_expense_categories()` → pick the article whose type matches the
   direction (payroll/materials for an outflow; a revenue article for income) →
   `expense_id`.
3. For a payout, resolve the master first in the `staff` category. Staff has **no
   name search** — `altegio_list_staff` takes only `page`/`count` (no `query`).
   List, then match the row whose `name` is "Marina" client-side → its `id` is the
   `staff_id`. If two staff share a name, ask which; don't guess.
4. **✍** `altegio_create_transaction(amount=5000, account_id=<1>,
   expense_id=<2>, staff_id=<3>, comment="Salary")` → `id`. This records the
   money out; it does not compute what is owed or pay through the register.

## Task: report or attribute transactions

"How much came through the Main register today?" / "all transactions for master
#678 in May" / "what has client #12345 paid?".

1. `altegio_list_transactions(account_id=<id>, start_date="2025-05-01",
   end_date="2025-05-31")` (dates are `YYYY-MM-DD`, inclusive) — or filter by
   `staff_id=678` / `client_id=12345` with the date range — paging to the end.
2. Sum each row's `amount`; group on the nested object — `expense.title` for an
   income/expense breakdown, `master.id`/`master.name` per staff,
   `client.id`/`client.name` per client. `total` is the record count, not a sum.

## Task: correct or remove an entry

Update sends only the fields you supply; omitted fields stay as-is, and empty
`comment`/`date` is rejected, not blanked: **✍**
`altegio_update_transaction(transaction_id=<id>, date="2026-06-10")`.

Delete is a **hard delete** that alters the audited ledger — no undo:

1. `altegio_get_transaction(transaction_id=<id>)` → verify it is a manual entry
   you created here, not a checkout/sale row. The list feed (`/transactions`)
   surfaces sale records that this delete path (`/finance_transactions`) does not
   own; deleting one won't reverse the sale, receipt, or inventory. If it is a
   real sale, stop and route to the register flow.
2. **✍** `altegio_delete_transaction(transaction_id=<id>, confirm=True)`.

## Pagination & limits

`page` (1-based) + `count` (default 50, max 200); page sequentially, not in
parallel (~5 req/sec / 200 req/min cap); stop when `next_cursor` is null.
`altegio_list_accounts` / `altegio_list_expense_categories` are NOT paginated and
are **per-salon and configurable** — never hardcode an ID. Defaults: registers
"Main" (cash) + "Settlement Account" (bank); ~12 expense articles
(material/goods purchases, payroll = expense; plus revenue articles). A 403 on a
finance endpoint means the connected `user_token` lacks the finances right
(re-grant at token creation), not a bad ID.

## Cross-category pointers

- `client_id` — `clients` category: `altegio_list_clients(query="+15551234567")`
  (name/phone/email search).
- `staff_id` (→ `master_id`) — `staff` category. Unlike clients, staff has **no
  name search**: `altegio_list_staff` paginates only; list and match `name`
  client-side (or use `altegio_get_staff` if you already hold the numeric id).
- `company_id` — `companies` category: `altegio_list_companies`; omit on any tool
  to use the connected default salon.
- Whether a visit should have produced a transaction (e.g. no-show
  `attendance=-1`) — read the record in the `appointments` category.
