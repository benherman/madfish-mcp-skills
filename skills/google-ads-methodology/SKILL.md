---
name: google-ads-methodology
description: >-
  Manage live Google Ads accounts through the Mad Fish MCP server — audit
  accounts, research keywords, build and structure campaigns, write responsive
  search ads, set bidding and budgets, add negative keywords, and pull
  performance reports. Use this skill whenever the user wants to inspect,
  analyze, create, or optimize Google Ads campaigns, ad groups, keywords, or
  budgets and a Mad Fish MCP connection (madfish-adops) is available. Triggers
  include "audit my Google Ads", "build a search campaign", "keyword research",
  "raise/lower my bids", "add negative keywords", "how are my campaigns
  performing", "pause this campaign", "create an RSA". This skill executes real
  changes against a live account — it pairs Google Ads domain expertise with the
  MCP's discovery meta-tools.
---

# Google Ads Methodology (MCP-connected)

Comprehensive Google Ads campaign management — account audits, keyword research,
campaign setup, responsive search ad (RSA) copy, bid and budget optimization,
negative keywords, and performance reporting. Unlike an advisory-only skill,
this one **acts on a live account** through the Mad Fish MCP server.

## How this skill executes work — the MCP discovery pattern

The Mad Fish MCP does not expose Google Ads operations as named tools directly.
It exposes **discovery meta-tools**, and you route every action through them:

1. `discover_search_operations(query, platform="google_ads")` — find the exact
   operation for a task. Returns operation `name` + a parameters schema.
2. `discover_execute_operation(name, args)` — run that operation with a JSON
   args object matching the schema.
3. `discover_list_platforms()` — confirm the tenant is licensed for
   `google_ads` and what capabilities (read / write / delete) the current user
   has.
4. `discover_help()` — workflow reminder if you get stuck.

**Always search first, then execute.** Operation names can change between
server versions — never hardcode them. Search with a 2–4 word query (action +
noun), copy the returned `name` verbatim into `discover_execute_operation`, and
build `args` from the returned schema.

Example round-trip:

```
discover_search_operations(query="list campaigns", platform="google_ads")
  → returns name: "list_campaigns_google_campaigns_get"
discover_execute_operation(
  name="list_campaigns_google_campaigns_get",
  args={"customer_id": "3268297149", "status": "ENABLED", "limit": 50}
)
```

## Capability awareness (read / write / delete)

Every Google Ads operation is one of three capability tiers, and the user's
subscription may not include all of them:

- **Read** — `list_*`, `get_*`, `lookup_*`, `generate_keyword_*`,
  `get_campaign_metrics`. Always available to any licensed seat.
- **Write** — `create_*`, `update_*`. Requires Write capability.
- **Delete** — `delete_*`, `remove_*`. Requires Delete capability.

If an execute call returns `"requires 'write' capability"` or
`"requires 'delete' capability"`, tell the user plainly which capability they're
missing on `google_ads` and that they can upgrade in the portal. Do NOT try to
work around it. Call `discover_list_platforms()` first if you're unsure what the
user is allowed to do before proposing a write-heavy plan.

## Reference conventions (memorize these)

- **Money is in micros.** 1 unit of account currency = 1,000,000 micros.
  $25.00/day = `25000000`. $2.00 CPC ceiling = `2000000`. Always convert
  dollars → micros before passing `*_micros` fields, and divide by 1,000,000
  when displaying `cost_micros` back to the user.
- **customer_id** is the 10-digit Google Ads account ID, no dashes
  (`3268297149`, not `326-829-7149`). Get it from
  `list_accounts` if the user doesn't provide it.
- **Match types:** `EXACT`, `PHRASE`, `BROAD`. Exact = tightest control,
  Broad = widest reach (use with Smart Bidding + negatives).
- **Campaign statuses:** `ENABLED`, `PAUSED`, `REMOVED`. New campaigns should
  default to `PAUSED` so nothing spends before the user reviews it.
- **Channel types:** `SEARCH`, `DISPLAY`, `PERFORMANCE_MAX`, `VIDEO`,
  `DISCOVERY`.
- **Bidding strategies:** `MANUAL_CPC`, `MAXIMIZE_CONVERSIONS`,
  `MAXIMIZE_CONVERSION_VALUE`, `TARGET_CPA`, `TARGET_ROAS`, `MAXIMIZE_CLICKS`.
  MANUAL_CPC only works for SEARCH or DISPLAY. TARGET_CPA needs
  `target_cpa_micros`; TARGET_ROAS needs `target_roas` (e.g. `3.5` = 350%).
- **New budgets are dedicated by default** — the MCP creates a non-shared
  campaign budget unless you pass `shared_budget: true`.

## Golden rule for writes

**Never create, edit, or delete without confirming the plan with the user
first.** Money is at stake. Before any write:

1. Summarize exactly what will change (campaign name, budget, bid strategy,
   keywords, etc.) in plain language with the dollar amounts spelled out.
2. Get an explicit "yes."
3. Create in `PAUSED` state where possible so the user can review in the Google
   Ads UI before it spends.
4. After the write, read back what you created (`get_campaign` /
   `get_campaign_metrics`) and confirm it matches intent.

---

## Workflow A — Account & campaign discovery (read)

Find the account and see what's running.

- List accounts: search `"list accounts"` → execute. Returns the accessible
  customer IDs and names.
- List campaigns: search `"list campaigns"` → execute with `customer_id` and
  optional `status` / `channel_type` / `limit`.
- Find one campaign by name: search `"lookup campaign by name"` → execute with
  `customer_id` + exact `name`.
- Get one campaign's detail: search `"get campaign"` → execute with
  `customer_id` + `campaign_id`.

Start here whenever the user is vague ("how are my ads doing?") — inventory the
account before proposing anything.

## Workflow B — Performance analysis & audit (read)

1. List ENABLED campaigns (Workflow A).
2. For each campaign of interest, search `"campaign performance metrics"` →
   execute `get_campaign_metrics` with `customer_id` + `campaign_id`. Returns
   impressions, clicks, cost_micros, conversions, conversion value (last 30
   days).
3. For keyword-level detail, search `"keyword performance metrics"` and
   `"list keywords"`.

**What to flag in an audit:**
- Campaigns spending with **zero conversions** over the window.
- Keywords with high cost and no conversions (candidates to pause or add as
  negatives).
- Budgets pacing at 100% every day (capped — may be leaving volume on the
  table) vs. barely spending (too narrow, or bids too low).
- Broad-match keywords with no negative-keyword list (waste risk).
- Legacy Manual CPC campaigns that could benefit from Smart Bidding if they have
  enough conversion history.

Present findings as a prioritized list: biggest wasted-spend items first, with
the specific dollar amounts (divide cost_micros by 1,000,000).

## Workflow C — Keyword research (read)

Uses the KeywordPlanIdeaService the MCP exposes.

- Generate ideas from seeds: search `"generate keyword ideas"` → execute with
  `customer_id` and a `request` object containing `seed_keywords`
  (or `seed_url` / `seed_site`), plus optional `geo_target_ids`
  (default US = `["2840"]`), `language_id` (default English = `"1000"`), and
  `include_monthly_volumes`.
- Get metrics for known keywords: search `"keyword historical metrics"` →
  execute with an explicit `keywords` list.

**How to use the output:** rank ideas by `avg_monthly_searches` and
`competition` (LOW / MEDIUM / HIGH). Favor high-volume + low-competition terms.
Group the shortlist into tight themes (see Workflow D) before building. Report
the top-of-page bid estimates (`low/high_top_of_page_bid_micros`) so the user
knows roughly what clicks will cost.

## Workflow D — Campaign structure & setup (write)

Good structure = tightly themed ad groups, each with a small set of closely
related keywords and 1–2 RSAs. Recommended hierarchy:

```
Campaign (one budget, one bidding strategy, one geo/language set)
└── Ad Group (one tight theme, e.g. "organic dog food")
    ├── Keywords (5–20, closely related, mixed match types)
    └── Ads (1–2 responsive search ads)
```

To build:

1. **Create the campaign** — search `"create campaign"` → execute with
   `customer_id` + a body: `name`, `channel_type: "SEARCH"`,
   `status: "PAUSED"`, `daily_budget_micros`, `bidding_strategy`, and any
   strategy-specific fields. Leave `shared_budget` off (dedicated budget).
2. **Set targeting** — search `"update campaign targeting"` → execute to add
   `language_codes` (e.g. `["en"]`) and `location_ids` (geo target IDs).
3. **Create ad groups** — search `"create ad group"` → execute per theme.
4. **Add keywords** — search `"create keywords"` → execute with the ad group's
   themed keyword list and match types (Workflow C output).
5. **Add the RSA** — Workflow E.

Confirm the whole structure with the user (campaign name, budget in dollars,
bidding strategy, ad groups + keyword themes) before creating anything. Build
PAUSED.

## Workflow E — Responsive search ad (RSA) copy (write)

RSAs take up to **15 headlines** (max 30 chars each) and **4 descriptions**
(max 90 chars each); Google mixes them automatically.

Copywriting guidance:
- Include the primary keyword in at least 3 headlines.
- Cover: the offer, a benefit, social proof, a CTA, and a differentiator.
- Pin sparingly — over-pinning kills Google's optimization. Pin only when legal
  or brand rules require a specific headline in position 1.
- Match the landing page's promise; misalignment tanks Quality Score.

Create via: search `"create ad"` (or `"create responsive search ad"`) →
execute with `customer_id`, `ad_group_id`, the headlines array, descriptions
array, and final URL. Read the schema the search returns for the exact field
names.

## Workflow F — Bid & budget optimization (write)

- **Change a daily budget:** search `"update campaign budget"` → execute with
  `customer_id`, `campaign_id`, `new_daily_budget` (dollars — this endpoint
  takes the dollar amount and converts internally).
- **Change bidding strategy:** search `"update campaign bidding"` → execute with
  the target strategy + its required field (`target_cpa_micros` /
  `target_roas` / `cpc_bid_ceiling_micros`).
- **Change a single keyword's bid:** search `"update keyword bid"` → execute
  with `cpc_bid_micros`.

**When to recommend which strategy:**
- New campaign, no conversion data → `MAXIMIZE_CLICKS` (with a CPC ceiling) or
  `MANUAL_CPC` to gather data.
- 15+ conversions/month → move to `MAXIMIZE_CONVERSIONS` or `TARGET_CPA`.
- Value-based goals (revenue/ROAS) with conversion values tracked →
  `TARGET_ROAS` or `MAXIMIZE_CONVERSION_VALUE`.

Never raise a budget or loosen a target without stating the new daily/monthly
spend implication in dollars.

## Workflow G — Negative keywords (write)

Prevents wasted spend on irrelevant queries. Essential for any Broad or Phrase
campaign.

- List existing negatives: search `"list negative keywords"` → execute.
- Add negatives: search `"create negative keywords"` → execute with `level`
  (`campaign` or `ad_group`), the corresponding `campaign_id` / `ad_group_id`,
  and a `keywords` list (text + match type).
- Remove negatives: search `"delete negative keywords"` → execute with the
  criterion resource names.

Sources for negatives: search-terms report patterns, obvious off-intent words
(free, cheap, jobs, DIY when you sell services, etc.), and competitor brand
terms if the user doesn't want to bid on them.

## Workflow H — Pausing, status changes & cleanup (write / delete)

- **Pause / enable a campaign:** search `"update campaign status"` → execute
  with `status: "PAUSED"` or `"ENABLED"`. This is the safe, reversible way to
  stop spend — prefer it over deletion.
- **Pause / enable a keyword:** search `"update keyword status"`.
- **Delete a campaign:** search `"delete campaign"` → execute (requires Delete
  capability). Deletion is effectively permanent (REMOVED state) — always
  confirm, and suggest pausing instead unless the user explicitly wants it gone.
- **Delete a keyword:** search `"delete keyword"`.

For cleanup requests ("clean up my account"), default to **pausing** low
performers and let the user decide what to delete. Never bulk-delete without an
itemized confirmation.

---

## End-to-end example session

User: "Build me a search campaign for organic dog food, $30/day, and research
keywords first."

1. `discover_list_platforms()` → confirm `google_ads` licensed + user has
   `write`.
2. `discover_search_operations(query="list accounts", platform="google_ads")`
   → execute → get `customer_id`.
3. Keyword research (Workflow C): `generate_keyword_ideas` with
   `seed_keywords: ["organic dog food"]`, `include_monthly_volumes: true`.
   Present the top 20 by volume + competition, grouped into 2–3 themes.
4. Propose the structure to the user: 1 SEARCH campaign, $30/day
   (`30000000` micros), `MAXIMIZE_CLICKS` to start, PAUSED; 3 ad groups by
   theme; 8–12 keywords each; 1 RSA per group. **Get confirmation.**
5. Create campaign (Workflow D) → set targeting → create ad groups → add
   keywords → add RSAs (Workflow E).
6. Add an initial negative list (Workflow G): `free`, `cheap`, `jobs`, `DIY`.
7. Read back with `get_campaign` and confirm. Tell the user it's PAUSED and how
   to enable it in the Google Ads UI (or offer to enable via
   `update_campaign_status`).

## Guardrails recap

- Search for the operation name every time; never assume it.
- Convert dollars ↔ micros correctly and always show dollars to the user.
- Confirm before every write; build PAUSED; read back after.
- Respect capability errors — surface the upgrade path, don't route around it.
- Prefer pause over delete for anything reversible.
