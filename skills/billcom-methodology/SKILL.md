---
name: billcom-methodology
description: >-
  Run real accounts-payable and accounts-receivable work in a live BILL
  (Bill.com) account through the Mad Fish MCP server — look up and manage bills,
  vendors, customers, and invoices; record vendor and customer payments; and
  read reference data (chart of accounts, bank accounts, items). Use this skill
  whenever the user wants to inspect, create, update, or delete anything in BILL
  / Bill.com and a Mad Fish MCP connection (madfish-adops) with the `billcom`
  platform is available. Triggers include "what bills are due in Bill.com",
  "enter a bill", "add a vendor", "create an invoice", "who owes us money",
  "record a payment", "pay this vendor", "list our customers", "what's in the
  chart of accounts". This skill executes real changes against a live financial
  account — it pairs BILL AP/AR knowledge with the MCP's discovery meta-tools.
---

# BILL (Bill.com) Methodology (MCP-connected)

Comprehensive BILL accounts-payable and accounts-receivable operations — bills,
vendors, customers, invoices, vendor credits, and payments — plus reference
data. Unlike an advisory-only skill, this one **acts on a live financial
account** through the Mad Fish MCP server. Because it moves money, the
confirmation discipline below is non-negotiable.

## How this skill executes work — the MCP discovery pattern

The Mad Fish MCP does not expose BILL operations as named tools directly. It
exposes **four discovery meta-tools**, and you route every action through them:

1. `discover_search_operations(query, platform="billcom")` — find the exact
   operation for a task. Returns operation `name` + a parameters schema.
2. `discover_execute_operation(name, args)` — run that operation with a JSON
   args object matching the schema.
3. `discover_list_platforms()` — confirm the tenant is licensed for `billcom`
   and what capabilities (read / write / delete) the current user has.
4. `discover_help()` — workflow reminder if you get stuck.

**Always search first, then execute.** Operation names follow the pattern
`billcom_{verb}_{noun}` (e.g. `billcom_list_bills`, `billcom_get_vendor`,
`billcom_create_invoice`, `billcom_record_ap_payment`), but never hardcode them
— search with a 2–4 word query, copy the returned `name` verbatim, and build
`args` from the returned schema.

Example round-trip:

```
discover_search_operations(query="list bills", platform="billcom")
  → returns name: "billcom_list_bills"
discover_execute_operation(
  name="billcom_list_bills",
  args={"max": 50, "filters": [{"field":"paymentStatus","op":"=","value":"1"}]}
)
```

## Capability awareness (read / write / delete)

Every BILL operation is one of three capability tiers, and the user's
subscription may not include all of them:

- **Read** — `billcom_list_*`, `billcom_get_*`. Always available to any
  licensed seat. All reference reads (chart of accounts, bank accounts, items,
  users, departments, locations, classes) are read-tier.
- **Write** — `billcom_create_*`, `billcom_update_*`,
  `billcom_record_ap_payment`, `billcom_record_ar_payment`, and the raw
  `billcom_crud_request` escape hatch. Requires Write capability. **Recording a
  payment is a money-moving write** — treat it with the most care.
- **Delete** — `billcom_delete_*` (bills, vendors, customers, invoices, vendor
  credits). In BILL, delete deactivates (soft-delete) the record. Requires
  Delete capability.

If an execute call returns `"requires 'write' capability"` or
`"requires 'delete' capability"`, tell the user plainly which capability they're
missing on `billcom` and that they can upgrade in the portal. Do NOT try to work
around it. Call `discover_list_platforms()` first if you're unsure what the user
is allowed to do before proposing a write-heavy plan.

## Reference conventions (memorize these)

- **Objects go in as an `obj`.** Create takes `obj` = the entity's fields (the
  router adds the BILL `entity` discriminator for you; do not set `id` on
  create). Update takes the record `id` in the path plus the changed fields in
  `obj`. Read and delete take just the `id`.
- **IDs are BILL's own opaque IDs** (e.g. `00n01ABCDEFGHIJKLMNO`). Get them from
  the matching `list_*` operation first — never invent them.
- **List = filter + paginate.** `billcom_list_{noun}` accepts `start`
  (0-based offset), `max` (page size, up to 999), `filters`, and `sort`.
  `filters` is a JSON array of `{"field": ..., "op": ..., "value": ...}`
  (ops include `=`, `<`, `>`, `<=`, `>=`, `!=`, `in`); `sort` is a JSON array of
  `{"field": ..., "asc": 0|1}`. Page with `start`/`max` for large result sets.
- **Money is a plain decimal** in BILL (e.g. `1250.00`), not micros/cents.
  State amounts to the user exactly as they'll post.
- **Dates are ISO** (`YYYY-MM-DD`): `dueDate`, `invoiceDate`, `processDate`.
- **Payments use dedicated endpoints, not create.** Record an AP (vendor)
  payment with `billcom_record_ap_payment` (a `data` payload: `vendorId`,
  `processDate`, and a `billPays` array of `{billId, amount}`). Record an AR
  (customer) payment received with `billcom_record_ar_payment` (`customerId` +
  an `invoicePays` array of `{invoiceId, amount}`). Include
  `chartOfAccountId` / `bankAccountId` where the org requires it — read those
  first from the reference lists.
- **Escape hatch.** For any BILL entity not covered by a typed operation, use
  `billcom_crud_request` (`op` = Create/Read/Update/Delete, `entity`, `data`)
  or `billcom_list_records` (`entity`, filters/sort). Reach for the typed
  operations first.

## Golden rule for writes — money edition

**Never create a bill/invoice, or record ANY payment, without explicit
confirmation.** This account moves real money. Before any write:

1. Summarize exactly what will happen — vendor/customer name, each bill/invoice
   ID and dollar amount, the total, the bank/account it hits, and the process
   date — in plain language.
2. Get an explicit "yes" that names the amount.
3. After the write, read back the record (`get_*` / `list_*`) and confirm it
   matches intent.

For payments specifically: never batch-pay or "pay everything" without an
itemized list the user approves line by line. If the user is vague ("pay the
Acme bills"), list the matching bills with amounts and due dates and ask them to
confirm which ones and how much.

---

## Workflow A — AP discovery: what do we owe? (read)

1. List open bills: search `"list bills"` → execute with `filters` for unpaid/
   approved status and `sort` by `dueDate`.
2. For a specific vendor, first `"list vendors"` (filter by name) to get the
   `vendorId`, then filter bills by it.
3. Get one bill's detail: `"get bill"` → execute with its `id`.

Present as a prioritized list: soonest due / overdue first, with vendor, amount,
and due date.

## Workflow B — Enter a bill (write)

1. Resolve the vendor: `"list vendors"` (or `"create vendor"` if new — confirm
   details first).
2. Optionally read `"list chart of accounts"` / `"list items"` for the expense
   coding.
3. Propose the bill (vendor, invoice number, amount in dollars, due date, GL
   coding) → **get confirmation** → `"create bill"` with the `obj`.
4. Read back with `"get bill"` and confirm.

## Workflow C — Pay vendors (write — money moves)

1. Identify the bills to pay (Workflow A). Show the user the itemized list with
   amounts and due dates.
2. Read `"list bank accounts"` to pick the funding account.
3. **Confirm line by line** — which bills, how much on each, from which account,
   on what process date.
4. Record the payment: `"record ap payment"` → execute with `vendorId`,
   `processDate`, `billPays: [{billId, amount}, ...]`, and the bank account.
5. Read back the affected bills to confirm they show paid.

## Workflow D — AR discovery: who owes us? (read)

1. List open invoices: `"list invoices"` → filter by unpaid status, sort by
   `dueDate`.
2. For a customer, `"list customers"` to get the `customerId`, then filter.
3. Detail: `"get invoice"` with its `id`.

Present overdue receivables first, with customer, amount, and days outstanding.

## Workflow E — Raise an invoice & receive payment (write)

1. Resolve the customer: `"list customers"` (or `"create customer"` — confirm
   first).
2. Propose the invoice (customer, number, line items, amounts, due date) →
   **confirmation** → `"create invoice"`.
3. When payment arrives, record it: `"record ar payment"` → execute with
   `customerId` and `invoicePays: [{invoiceId, amount}, ...]`.
4. Read back the invoice to confirm it's marked paid.

## Workflow F — Reference & setup data (read)

Fetch these to build valid records or answer setup questions:
`"list chart of accounts"`, `"list bank accounts"`, `"list items"`,
`"list departments"`, `"list locations"`, `"list accounting classes"`,
`"list users"`. All read-tier.

---

## End-to-end example session

User: "What do we owe Acme Supplies, and pay the two oldest bills from our
operating account."

1. `discover_list_platforms()` → confirm `billcom` licensed + user has `write`.
2. `list vendors` (filter name "Acme Supplies") → get `vendorId`.
3. `list bills` (filter by that vendor + unpaid, sort by `dueDate` asc) →
   present the list with amounts and due dates.
4. `list bank accounts` → identify the operating account's id.
5. Show the user the two oldest bills with exact amounts and the funding
   account. **Get explicit confirmation naming the total.**
6. `record ap payment` with `vendorId`, `processDate`, both `billPays` lines,
   and the bank account id.
7. Read the two bills back → confirm paid. Report the total paid and remaining
   balance to the vendor.

## Guardrails recap

- Search for the operation name every time; never assume it.
- Resolve IDs via `list_*` before referencing them.
- Confirm before every write; for payments, confirm line-by-line with the total
  named; read back after.
- Never move money on a vague instruction — itemize and get sign-off.
- Respect capability errors — surface the upgrade path, don't route around it.
- Remember `delete` deactivates the record; still confirm before doing it.
