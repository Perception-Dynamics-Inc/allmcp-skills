---
name: allmcp-google-ads
description: Manage Google Ads through the AllMCP hub — campaign performance, budgets, keywords, ads, conversions, and Google's own recommendations. Use when the user asks to read campaign metrics, adjust budgets or keywords, create/pause campaigns, or upload offline conversions via AllMCP. NOT for Google Analytics, other ad platforms (Meta, TikTok, LinkedIn), or direct Google Ads API integration with your own developer token outside AllMCP.
---

# Google Ads via AllMCP

Through AllMCP your agent gets ~40 namespaced Google Ads tools
(`google_ads_list_campaigns`, `google_ads_run_gaql`, …) across 12 categories.
Platform mechanics (endpoint, connecting, hidden tools, quota) live in the
`allmcp` skill — install it alongside this one.

**This provider spends real money.** A budget bump, a status flip, an applied
recommendation — each changes live ad spend the moment it lands. Confirm with
the user before every ✍ step (name the exact campaign/amount you're about to
touch), and read the state back afterwards — report what the account now
shows, not what the mutation call claimed.

## Connect

- Provider key: `google_ads` (exactly this, snake_case).
- OAuth2: call `connect_provider(provider_key="google_ads")` with nothing
  else. The response is `action_required` with a Google consent URL — hand
  that URL to the user and stop. They approve in a browser; the connection
  completes on its own and AllMCP keeps the token refreshed.
- Never ask the user for a developer token, client secret, or refresh token —
  AllMCP owns all of that. If a call that worked before starts failing with an
  auth error, re-run `connect_provider` and let the user re-approve once.

## Model note — read before acting

- **Customer IDs, not "my account".** Every tool takes a 10-digit
  `customer_id` (dashes optional — `415-987-1052` and `4159871052` both
  work). Start at `google_ads_list_accessible_customers()`; if it returns
  several, ask which one — don't guess. A manager (MCC) login reaches child
  accounts via `login_customer_id=<manager's ID>` on the call; map the tree
  with `google_ads_list_customer_hierarchy` first.
- **All money is micros.** 1,000,000 micros = 1 unit of the *account*
  currency. A "$50/day budget" is `budget_micros=50_000_000`; a
  `cost_micros` of `12340000` is 12.34. Check the currency with
  `google_ads_get_customer` before reporting figures — micros are not
  automatically dollars.
- **Every campaign hangs off a budget resource.** `google_ads_create_campaign`
  creates a dedicated budget from its `budget_micros` param, but
  `google_ads_update_campaign_budget` mutates the *budget the campaign
  points at* — if that budget is shared, every attached campaign moves.
- **REMOVED is forever.** There is no hard delete: `google_ads_delete_campaign`
  sets status `REMOVED`, which is terminal and irreversible. "Pause" means
  `google_ads_update_campaign_status(..., status="PAUSED")` — reversible.
  Never reach for REMOVED when the user said pause.
- **New campaigns start PAUSED by default** (a safety net — nothing spends
  until you explicitly set `ENABLED`), and mutating tools accept
  `dry_run=True` for a validate-only pass. Use both: create paused, verify
  the setup, then enable with the user's go-ahead.
- **GAQL is the read language.** Anything the canned metrics tools don't
  cover goes through `google_ads_run_gaql` with a SQL-like query. `LIMIT N`
  in the query is the only result cap — there is no page-size parameter.

## Categories

`campaigns` and `metrics` are visible immediately after connecting; the rest
unlock via `describe_category("google_ads", <category>)` — call it before
first use of a category this session (it also carries the tool list).

| Category | Covers |
|---|---|
| `accounts` | Account discovery, customer hierarchy, login customer |
| `campaigns` | Campaign CRUD plus GAQL reads |
| `ad_groups` | Ad group and ad creative CRUD |
| `keywords` | Keyword and negative-keyword targeting |
| `budgets_bidding` | Campaign budgets and bidding strategies |
| `metrics` | GAQL reads on performance metrics (the core read surface) |
| `conversions` | Conversion actions plus offline conversion uploads |
| `diagnostics` | AI-driven diagnose_account, recommendations, apply |
| `assets` | Sitelinks, callouts, structured-snippet creative assets |
| `audiences` | User-list audiences and audience-targeting application |
| `pmax` | Performance Max channel reporting and gender exclusions |
| `demand_gen` | Demand Gen image assets, logo upload, asset-group linking |

## Task: Find which campaigns burned the budget this week

Pure reads — no confirmation needed, but get the account and currency right
before quoting numbers.

1. `google_ads_list_accessible_customers()` → e.g. `customer_id: "4159871052"`.
   More than one account? Ask the user which — a spend report against the
   wrong account is worse than no report.
2. `google_ads_get_customer(customer_id="4159871052")` → `currencyCode: "USD"`,
   timezone. Now micros ÷ 1,000,000 has a unit.
3. `google_ads_get_campaign_metrics(customer_id="4159871052",
   date_range="LAST_7_DAYS")` → per-campaign clicks, impressions,
   `cost_micros`, conversions. Sort by `cost_micros` yourself, or when you
   need a custom cut, `google_ads_run_gaql` with
   `SELECT campaign.name, metrics.cost_micros FROM campaign WHERE
   segments.date DURING LAST_7_DAYS ORDER BY metrics.cost_micros DESC LIMIT 10`.
4. Report cost in the account currency, and flag campaigns with high spend
   and zero conversions — that's the "burned" list the user actually wants.

## Task: Pause the underperformer safely

1. `google_ads_get_campaign(customer_id="4159871052", campaign_id=21473650984)`
   → confirm the name and current status match the campaign the user means.
   Pausing the wrong campaign silently kills a healthy revenue stream.
2. Show the user the campaign name, its recent spend, and the exact action
   ("pause 'Search — Brand US', spent $412.18 last 7 days — confirm?").
   Only on an explicit yes:
   **✍** `google_ads_update_campaign_status(customer_id="4159871052",
   campaign_id=21473650984, status="PAUSED")`.
3. Read it back: `google_ads_get_campaign(...)` → `status: "PAUSED"`. That
   read, not the mutation's success message, is what you report. PAUSED is
   reversible (`status="ENABLED"` turns it back on); do not "clean up" a
   paused campaign with REMOVED unless the user explicitly asks to remove it.

## Boundaries

- **No editing live ad copy.** Ads can be created
  (`google_ads_create_responsive_search_ad`), listed, and paused/enabled
  (`google_ads_update_ad_status`) — existing headlines/descriptions can't be
  rewritten. The Google Ads pattern is: create the revised ad, pause the old.
- **Keywords are add/list only** — no tool removes a keyword or edits its
  bid. Negatives can be added, not retracted.
- **Asset creation covers sitelinks** (`google_ads_create_sitelink_asset`);
  other asset types are list-only here.
- **Audiences are list-and-apply** — no creating user lists or uploading
  customer-match data.
- **No billing or payments surface** — invoices, payment methods, and account
  budgets orders live in the Google Ads UI only.
- **No account administration** — can't create customer accounts, link
  managers, or manage user access.
