---
name: workamajig-methodology
description: >-
  Run real work in a live Workamajig agency-management account through the Mad
  Fish MCP server — inspect and manage projects, campaigns, tasks, timesheets,
  contacts and companies, estimates and purchase orders, billing and invoices,
  and pull financial reports (Client/Project P&L, Trial Balance, AR/AP aging,
  General Ledger). Use this skill whenever the user wants to look up, create,
  update, or delete anything in Workamajig and a Mad Fish MCP connection
  (madfish-adops) with the `workamajig` platform is available. Triggers include
  "look up a project in Workamajig", "create a project", "who's on this account",
  "log time", "approve timesheets", "create an estimate", "raise a client
  invoice", "what's our AR aging", "pull the P&L for this client", "add a
  vendor", "cut a PO". This skill executes real changes against a live account —
  it pairs Workamajig domain knowledge with the MCP's discovery meta-tools.
---

# Workamajig Methodology (MCP-connected)

Comprehensive Workamajig agency operations — projects and campaigns, tasks and
time, CRM (contacts/companies/leads/opportunities), estimates and purchasing,
billing and payments, and financial reporting. Unlike an advisory-only skill,
this one **acts on a live account** through the Mad Fish MCP server.

## How this skill executes work — the MCP discovery pattern

The Mad Fish MCP does not expose Workamajig operations as named tools directly.
It exposes **four discovery meta-tools**, and you route every action through
them:

1. `discover_search_operations(query, platform="workamajig")` — find the exact
   operation for a task. Returns operation `name` + a parameters schema.
2. `discover_execute_operation(name, args)` — run that operation with a JSON
   args object matching the schema.
3. `discover_list_platforms()` — confirm the tenant is licensed for
   `workamajig` and what capabilities (read / write / delete) the current user
   has.
4. `discover_help()` — workflow reminder if you get stuck.

**Always search first, then execute.** Operation names follow the pattern
`workamajig_{verb}_{noun}` (e.g. `workamajig_get_projects`,
`workamajig_create_project`, `workamajig_get_client_pnl`), but never hardcode
them — search with a 2–4 word query (action + noun), copy the returned `name`
verbatim, and build `args` from the returned schema.

Example round-trip:

```
discover_search_operations(query="get projects", platform="workamajig")
  → returns name: "workamajig_get_projects"
discover_execute_operation(
  name="workamajig_get_projects",
  args={"searchFor": "Acme rebrand", "searchField": "projectname"}
)
```

## Capability awareness (read / write / delete)

Every Workamajig operation is one of three capability tiers, and the user's
subscription may not include all of them:

- **Read** — `workamajig_get_*`, `workamajig_search_*`, `workamajig_list_*`.
  Always available to any licensed seat. All the financial reports (P&L, Trial
  Balance, AR/AP, GL) are read-tier.
- **Write** — `workamajig_create_*`, `workamajig_update_*`, plus the timesheet
  `complete`/`reject` and journal-entry actions. Requires Write capability.
- **Delete** — `workamajig_delete_*` (projects, tasks, contacts, companies,
  vendors, campaigns, deliverables, timesheets, time entries, to-dos,
  opportunities, leads, employees, purchase orders). Requires Delete capability.

If an execute call returns `"requires 'write' capability"` or
`"requires 'delete' capability"`, tell the user plainly which capability they're
missing on `workamajig` and that they can upgrade in the portal. Do NOT try to
work around it. Call `discover_list_platforms()` first if you're unsure what the
user is allowed to do before proposing a write-heavy plan.

## Reference conventions (memorize these)

- **Records come back and go in as arrays.** Create/update operations take a
  `records` argument that is a single object OR a list of objects (max 100 per
  batch). Deletes take a `keys` array of key objects, e.g.
  `[{"key": "abc123"}]`.
- **Everything is addressed by a Key**, not a numeric ID: `projectKey`,
  `contactKey`, `companyKey`, `vendorKey`, `campaignKey`, `timesheetKey`,
  `activityKey`, `conversationKey`, `employeeKey`. Get keys from the matching
  `get_*` / `search_*` operation first — don't guess them.
- **Search vs. get.** `workamajig_get_{noun}` fetches by key (or lists when no
  key is given); `workamajig_search_{noun}` does a text search (contacts,
  companies, vendors, employees, leads, activities, conversations) and accepts
  `text`, `getFavorites`, `getRecents`, `dateUpdated`.
- **Projects read is rich.** `workamajig_get_projects` accepts `searchFor` +
  `searchField` (`projectnumber` / `projectname` / `companyname` /
  `campaignname` / `description`) and `include*` flags
  (`includeTeam`, `includeTasks`, `includeTodos`, `includeMiscCosts`,
  `includeEstimates`, `includeSpecSheets`, `all`) to pull related datasets in
  one call.
- **Dates are ISO** (`YYYY-MM-DD`) on every date filter (`startdate`, `enddate`,
  `startDate`, `endDate`, `dateUpdated`).
- **Reference lists are read-only lookups** you'll need to build valid records:
  `workamajig_get_project_types`, `_project_status`, `_services`, `_teams`,
  `_titles`, `_offices`, `_departments`, `_payment_terms`, `_payment_methods`,
  `_sales_tax`, `_gl_accounts`, `_gl_classes`, `_gl_companies`. Fetch the
  relevant one before creating an entity that references it.

## Golden rule for writes

**Never create, edit, or delete without confirming the plan with the user
first.** Client money, invoices, and payments are at stake. Before any write:

1. Summarize exactly what will change (project name/number, client, dates,
   estimate/invoice amounts in dollars, who's assigned) in plain language.
2. Get an explicit "yes."
3. After the write, read back what you created/changed (the matching `get_*`)
   and confirm it matches intent.

Prefer status changes / updates over deletes for anything reversible. Never
bulk-delete without an itemized confirmation.

---

## Workflow A — Account, client & project discovery (read)

Inventory before acting.

- Find a client/company: search `"search companies"` → execute with `text`.
- Find a project: search `"get projects"` → execute with `searchFor` +
  `searchField`, or list. Add `includeTeam`/`includeTasks` for detail.
- Find a campaign: search `"get campaigns"` → execute (optionally `campaignKey`).
- Find a person: search `"search contacts"` / `"search employees"`.

Start here whenever the user is vague ("what's happening on the Acme account?")
— pull the company, its projects, and their status before proposing anything.

## Workflow B — Project & campaign setup (write)

1. Gather references: `get_project_types`, `get_project_status`,
   `get_services`, and the client's `companyKey` (Workflow A).
2. Create the campaign if the work is campaign-level: search
   `"create campaign"` → execute.
3. Create the project: search `"create project"` → execute with a `records`
   object (name, number, client/company key, project type, dates). Confirm the
   name, client, and dates with the user first.
4. Add deliverables / tasks: `"create deliverable"`, `"create task"`.
5. Read back with `"get projects"` (`includeTasks: true`) and confirm.

## Workflow C — Tasks, time & approvals (write)

- See a project's tasks: search `"get tasks"` → execute with `projectKey`.
- Log time: search `"create time entry"` → execute with the timesheet/task key
  and hours.
- Review timesheets: search `"get timesheets"` → execute with `userKey` /
  `status` / date range.
- Approve: search `"complete timesheet"` → execute with the timesheet key(s).
  Reject: search `"reject timesheet"`. Both are write-tier and affect payroll/
  billing — confirm which timesheets before acting.

## Workflow D — CRM: contacts, companies, leads & opportunities (read/write)

- Create/update a company or contact: `"create company"` / `"create contact"` /
  `"update contact"`. Fetch `get_company_types` and `get_sources` first for
  valid values.
- Manage the pipeline: `"get opportunities"` / `"list opportunities"`,
  `"create opportunity"`, `"update opportunity"`; reference
  `get_opportunity_stages` / `_status` / `_outcomes`. Leads: `"search leads"`,
  `"create lead"`.

## Workflow E — Estimates & purchasing (write)

- Build an estimate: search `"create estimate"` → execute; read rate sheets
  first (`get_service_rate_sheets`, `get_item_rate_sheets`).
- Cut a purchase order: `"create purchase order"` → execute, then
  `"create purchase item"` for line items. Confirm vendor and amounts in dollars.
- Media planning: `"create media plan"`, `"get media plans"`,
  `"create insertion order"`.
- Receipts against POs: `"create receipt"`.

## Workflow F — Billing, invoices & payments (write)

Money-moving territory — confirm every amount in dollars before executing.

- Client invoices: `"get client invoices"`, `"create client invoice"`,
  `"update client invoice"`.
- Client payments: `"create client payment"`; void with
  `"void client payment"` (confirm the check/payment first).
- Vendor invoices & payments: `"create vendor invoice"`,
  `"create vendor payment"`.
- Expenses: `"create expense report"` + `"create expense report item"`.
- Journal entries: `"create journal entry"` — the journal number auto-assigns
  (last + 1); use `"get last journal number"` to preview the next number.

## Workflow G — Financial reporting (read)

All read-tier. Present dollar figures clearly and cite the report + date range.

- Client P&L: search `"get client pnl"` → execute with `client` (+ optional
  date range / `includeChildren`).
- Project P&L: `"get project pnl"`.
- Trial Balance: `"get trial balance"` (optionally filter `accountNumbers`).
- Balance Sheet / P&L / General Ledger: `"get balance sheet"`,
  `"get profit and loss"`, `"get general ledger"` — each accepts
  `glCompanies` / `accountingClass` / `cashBasis` filters.
- AR / AP aging: `"get accounts receivable"` (by `client`) /
  `"get accounts payable"` (by `vendor`), with optional `agingPeriod`.
- Look up specific GL accounts with balances: `"lookup gl accounts"`.

When a user asks "how are we doing on this client," combine the Client P&L with
AR aging so they see profitability and what's still owed in one answer.

---

## End-to-end example session

User: "Set up a new project for Acme's Q3 campaign, then show me Acme's P&L and
what they still owe us."

1. `discover_list_platforms()` → confirm `workamajig` licensed + user has
   `write`.
2. `search companies` (`text="Acme"`) → get the `companyKey`.
3. `get_project_types` + `get_project_status` for valid values.
4. Propose the project (name, client, type, dates) → **get confirmation** →
   `create project`. Read back with `get projects`.
5. `get client pnl` (`client="Acme"`) → present profitability.
6. `get accounts receivable` (`client="Acme"`) → present the aging balance.
7. Summarize: project created (number X), Acme YTD margin $Y, $Z outstanding
   (N days overdue).

## Guardrails recap

- Search for the operation name every time; never assume it.
- Resolve Keys via `get_*`/`search_*` before referencing them.
- Confirm before every write; spell out dollar amounts; read back after.
- Respect capability errors — surface the upgrade path, don't route around it.
- Prefer status changes / updates over deletes for anything reversible; never
  bulk-delete without itemized confirmation.
