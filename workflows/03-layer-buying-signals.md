# Layer buying + timing signals

**Goal:** Add timing signals on top of your core company net so the TAM is sorted by *who is ready to buy now* — funding, intent, news, headcount growth, active hiring, lookalike, and Google-ranking discovery.

## When to use

Use this after you have a working core net from the previous step (industry + keywords + geo + size returning a sane `total_count`). Timing signals are *layers*, not a starting point — you add them to an existing `/search-company` filter set to slice the same TAM into "reach out this week" vs. "nurture" tiers. Reach for them when you need to prioritize a large flat list, catch accounts at a trigger moment (fresh raise, exec change, hiring spike), or find net-new lookalikes of your best customers.

## Prompt for Claude Code

```
Using the Prospeo MCP server (PROSPEO_API_KEY=your_key_here), start from my
existing core /search-company filter set and layer timing signals on top.

Add these one at a time, running a count-first page-1 request after each so I
can see how much of the TAM each signal captures:

1. company_funding — stage ["Series A","Series B"], funding_date 365 (raised in
   the last year). Recent raise = budget + expansion.
2. company_intent — active_research / in_depth_research on my dashboard topic_ids
   (ask me for the topic_id; it is not enumerable via API).
3. company_news — categories ["Leadership Change","Expansion",
   "Mergers & Acquisitions"], timeframe_days 180.
4. company_headcount_growth — timeframe_month 12, min 15, departments ["Sales"].
5. company_job_posting_hiring_for — boolean_search "(engineer OR developer) AND
   senior AND !intern".
6. company_lookalike — domain mode against my best-customer domain, minimum_tier T2.
7. company_google_discovery — seo_keywords for the category I sell into (Pro key).

For each, report pagination.total_count before and after, note the plan tier,
and flag any PLAN_REQUIRED so I can switch to the Pro key. Do NOT enrich yet.
```

## Prospeo filters / endpoints used

All of these are **company filters** on `POST /search-company` (and can also be added to `/search-person` to constrain which companies the people work at). Host `https://api.prospeo.io`, auth header `X-KEY`.

| Filter | Plan | Key params (exact names) | What it does / when to use |
|---|---|---|---|
| `company_funding` | Starter+ | `stage`, `funding_date` (days since last raise), `last_funding` `{min,max}`, `total_funding` `{min,max}`, `investors`, `was_in_accelerator`, `accelerator_name` | Filter by round + recency + amount. `funding_date:365` = raised in the last 365 days = fresh budget. `stage` uses the exact Funding-stage enum (`Series A`, `Series B`, `Seed`, `Pre seed`…); `last_funding`/`total_funding` use the Revenue-range enum strings (`1M`,`10M`,`100M`…). Use to catch the post-raise expansion window. |
| `company_intent` | Free* | `topic_ids` (1–30), `early_research`, `active_research`, `in_depth_research` | Buying-intent stage on topics you configure in the Prospeo **dashboard** Intent settings. `topic_ids` are **NOT enumerable via API** — you must paste them in. Use `active_research`/`in_depth_research:true` for the hottest accounts. Strongest single "buying now" signal. |
| `company_news` | Free | `keywords`, `categories`, `timeframe_days` | Recent company news by keyword/category. Valid `categories`: Funding & Investment, Mergers & Acquisitions, Product Launch, Partnership, Expansion, Layoffs & Restructuring, IPO, Leadership Change, Legal & Regulatory, Awards & Recognition. Use `Leadership Change`/`Expansion` as a trigger to reach out with a timely reason. |
| `company_headcount_growth` | Free | `timeframe_month`, `min`, `max` (−100..10,000%), `departments` | Headcount growth % overall or per department over N months. `departments` uses the **separate Headcount-Growth enum** (Sales, Technical, Marketing, Operations…). Use `{"timeframe_month":12,"min":15,"departments":["Sales"]}` to find teams scaling a function you sell into. |
| `company_job_posting_hiring_for` | Starter+ | `include`/`exclude`/`match_type` **or** `boolean_search` | Titles the company is actively hiring for = live hiring-intent. `boolean_search` supports `AND`/`OR`/`!exclude`/`()` grouping (e.g. `(engineer OR developer) AND senior AND !intern`). Use when the roles they're hiring for imply they need what you sell. |
| `company_lookalike` | Starter+ | one of: `company_oids`, `icp_text`, `person_oids` (`/search-person` only), `domain`; plus `minimum_tier` (T1/T2/T3), `match_all`, `same_language` | AI similarity to a seed. Use exactly one seed mode. `domain` mode against a best-customer domain, `minimum_tier:T1` = most similar / T3 = broadest (default). Use to find net-new accounts that resemble your closed-won, not to re-filter firmographics. |
| `company_google_discovery` | **Pro+** | `seo_keywords` (1–100) | Finds companies by the SEO keywords they **rank for** on Google (via `/search-suggestions company_google_discovery_search`). Use to surface a competitive/adjacent set by the terms they win on organic search. Requires the Pro key or returns `PLAN_REQUIRED`. |

**Supporting endpoints:**
- `POST /search-suggestions` — **FREE, 0 credits.** Resolve verbatim values before filtering: `company_funding_investors_search` (investors/accelerators), `company_google_discovery_search` (SEO keywords), plus news keywords. Exactly one `*_search` param per call, ≥2 chars.
- `GET /account-information` — **free.** Pre-flight `remaining_credits` before you start paginating.

> Note: `company_intent` topics and `company_key_execs` (Pro+, exec appointments/departures) are two more timing levers configured in-dashboard; `company_key_execs` uses `event_types` + `timeframe_days` and is Pro-gated.

## Steps

1. **Start from the core net.** Take your existing `/search-company` filter object from the previous step and record its baseline `pagination.total_count` from a page-1 request.
2. **Resolve any non-enumerable values via `/search-suggestions`** (free): investor names for `company_funding.investors`, SEO keywords for `company_google_discovery.seo_keywords`. For `company_intent`, grab your `topic_ids` from the Prospeo dashboard Intent settings — the API cannot list them.
3. **Add one signal at a time and count.** Merge a single timing filter into the core net, POST page 1, and read `pagination.total_count`. Measure each signal's capture rate against the baseline before combining — layering signals is AND logic and shrinks the set fast.
4. **Watch for `PLAN_REQUIRED`.** `company_google_discovery` (and `company_key_execs`) are Pro-gated. On HTTP 400 `PLAN_REQUIRED`, the `filter_error` names the rejected filter and its min plan; switch to the Pro-tier key and retry.
5. **Tier the TAM.** Tag each account with which signals it matched (e.g. `funding+intent` = P1, `hiring` only = P2). Keep the union so you can prioritize without dropping accounts.
6. **Paginate the priority tiers first.** Loop `page` 1 → `total_page` (max 1,000 pages, 25/page ⇒ 25,000 result cap) on the tightest, highest-intent segment; throttle from `x-minute-request-left` / `x-daily-request-left`, back off on `429`.
7. **Dedupe by `domain`** (root domain on the company object), then hand the qualified, signal-tagged list to the persona/enrich step.

## Credits

Search costs **1 credit per page that returns ≥1 result** (25 results/page). Layering signals with count-first page-1 requests is effectively free scouting — each count is 1 credit, and identical result pages already pulled within **30 days** come back `"free": true` at no charge. `/search-suggestions` and `GET /account-information` are **0 credits**. Adding these filters does not cost extra — you pay per page pulled, and a tighter (higher-intent) net means fewer pages. No enrichment happens in this step.

## Output

A signal-tagged, domain-deduped company list where each account carries the timing signals it matched (funding / intent / news / headcount-growth / hiring / lookalike / google-discovery), ready to be sorted into outreach priority tiers before the persona and enrichment steps.
