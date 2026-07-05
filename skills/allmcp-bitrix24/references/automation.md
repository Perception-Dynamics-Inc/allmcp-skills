# Bitrix24 automation — workflows (Business Processes)

Start a Bitrix24 **Business Process** (workflow template) on a record, and
list the instances running right now — that's the whole surface. "Automation
rules" (robots — trigger rules on pipeline stages) have no tools here; don't
offer them. **✍** = real launch; confirm template and target if vague.

## Model note — read before acting

- Both tools are admin-only and need the `bizproc` scope — `ACCESS_DENIED`
  means the webhook owner isn't an administrator, not bad IDs. REST start
  additionally requires a paid or demo plan.
- You cannot pass input values to a template — the API's `PARAMETERS` is not
  exposed.
- For CRM records the third `document_id` element is **prefixed** —
  `"DEAL_42"`, never the bare `'42'` of the tool's own examples. Table below.
- `bitrix24_list_workflows` rows carry only `ID`, `MODIFIED`, `OWNED_UNTIL` —
  template, document, and starter are invisible. Never promise to find a
  workflow "by name"; match only IDs returned by `bitrix24_start_workflow`.
- `include` changes nothing in this category — leave it at the default.

## Task: start a workflow on a CRM record

1. Get `template_id` from the user. A template is bound to one document type —
   a deal template cannot start on a lead.
2. Resolve the record's numeric ID via the `crm` category:
   `bitrix24_list_deals(query="Acme")` → `ID` 42 (same pattern:
   `bitrix24_list_leads`, `bitrix24_list_contacts`, `bitrix24_list_companies`).
3. **✍** `bitrix24_start_workflow(template_id=7, document_id=["crm",
   "CCrmDocumentDeal", "DEAL_42"])` → string instance ID (`"66e81a64…"`).
4. The returned ID is the confirmation — report it. Don't verify via the
   list: rows can't be tied to a document; finished workflows vanish anyway.

## Task: check what's running / find a stuck workflow

1. `bitrix24_list_workflows()` → rows of `ID`/`MODIFIED`/`OWNED_UNTIL`, most
   recently modified first; `total` = count of running instances.
2. Stuck signal: `OWNED_UNTIL` more than ~5 minutes behind the current time.
   `MODIFIED` is last activity, not start time — it cannot tell you how long
   an instance has been running.
3. Report instance IDs and staleness, and say plainly that naming which
   process an instance is needs the Bitrix24 UI.

## Reference — `document_id` by target record

CRM third elements are prefixed; list elements use their plain numeric ID
(resolve via the `lists` category). Class strings carry single backslashes in
the final string value — escape per your serialization (`\\` in JSON); the
tool's own lists example is wrong on both backslashes and class name.

| Target | `document_id` |
|---|---|
| CRM deal | `["crm", "CCrmDocumentDeal", "DEAL_42"]` |
| CRM lead | `["crm", "CCrmDocumentLead", "LEAD_88"]` |
| CRM contact | `["crm", "CCrmDocumentContact", "CONTACT_7"]` |
| CRM company | `["crm", "CCrmDocumentCompany", "COMPANY_15"]` |
| Smart process (SPA) item | `["crm", "Bitrix\Crm\Integration\BizProc\Document\Dynamic", "DYNAMIC_147_1"]` |
| Universal list element | `["lists", "Bitrix\Lists\BizprocDocumentLists", "123"]` |
