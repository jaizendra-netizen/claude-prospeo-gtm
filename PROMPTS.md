# Prospeo Pipeline Prompts

Six ready-to-paste prompts for **Claude Code** (with the Prospeo MCP server + a Prospeo API key configured). Each runs one stage of a TAM-to-outreach pipeline against the Prospeo API.

**Before you start:**

- Set your key: `PROSPEO_API_KEY=your_key_here` (Prospeo auth is the header `X-KEY: <api_key>`, **not** `Authorization: Bearer`). Host: `https://api.prospeo.io`.
- Replace every `<<< ... >>>` placeholder with your ICP details before pasting.
- Prospeo credit facts to keep in mind: `/search-suggestions` and `/account-information` are **free (0 credits)**; search is **1 credit per page** that returns ≥1 result (25 results/page); enrich-person is **1 credit per email found** (`enrich_mobile` = 10 credits per mobile); re-running the exact same search page within **30 days** is free, and re-enriching the same record within **90 days** is free.
- The `prospeo` MCP exposes only `search` + `enrich` + `account`. It does **not** expose `/search-suggestions` or the `/bulk-*` endpoints — those stages use raw REST (`curl`) as noted in each prompt.

---

### 1. TAM Builder — resolve values, count-first, then paginate a multi-signal company net

```
You have the Prospeo MCP server and PROSPEO_API_KEY (header: X-KEY, host https://api.prospeo.io). Build a company TAM.

MY ICP:
<<< describe target companies: industry / sub-niche, HQ geo, size band, revenue, public/private, funding stage, and the definition edge — e.g. "cybersecurity vendors HQ'd in US/UK/Canada, 50–1000 employees, that sell to enterprise security teams" >>>

Do this in order:

1. RESOLVE VALUES FIRST (free, 0 credits). Use raw REST to POST /search-suggestions with exactly ONE *_search param per call (min 2 chars) to translate my human phrases into Prospeo's canonical stored strings. Resolve at least:
   - location_search  → canonical HQ locations (e.g. "United States", "United Kingdom")
   - industry_search  → exact company_industry enum values
   - technology_search / company_funding_investors_search — only if my ICP names a stack or investor
   Print the resolved values you'll use verbatim. Do NOT pass raw strings like "nyc" — Prospeo rejects non-verbatim values.

2. COUNT-FIRST (free to read). POST /search-company page 1 with a MULTI-SIGNAL net (never company_industry alone — it undercounts). Combine:
   - company_location_search { include: [...resolved...] }
   - company_industry { include: [...resolved...] }
   - company_keywords { include: [<<< 6–10 ICP keywords >>>], search_everywhere: false, sources: ["specialties","seo_description","ai_description","website_pages"] }
   - company_headcount_range or company_headcount_custom for my size band
   Read pagination.total_count and total_page. Report the TAM size and estimated search-credit cost (total_page × 1) BEFORE pulling. Also GET /account-information to confirm remaining_credits covers it.

3. PAGINATE. Loop page 1 → total_page (hard cap 1000 → max 25,000 results). Collect results[].company into a local NDJSON file. Throttle using the x-minute-request-left / x-daily-request-left response headers; back off and wait x-minute-reset-seconds on any 429.

4. If total_count > 25,000, split into non-overlapping sub-searches (by geo or size band), pull each, then union.

5. DEDUPE BY DOMAIN (root domain field, not name/website). Save the deduped company list locally and report the final count.
```

---

### 2. Free Market Sizing — size the TAM (and useful sub-segments) for zero credits

```
You have the Prospeo MCP + PROSPEO_API_KEY (X-KEY header, host https://api.prospeo.io). Size my market WITHOUT pulling records — reading pagination.total_count costs nothing.

MY ICP:
<<< describe target companies: industry, geo, size band, revenue, funding — same shape as the TAM Builder prompt >>>

Do this:

1. GET /account-information first (free) and report remaining_credits, used_credits, next_quota_renewal_days.

2. Resolve any human phrases to canonical values via /search-suggestions (free, 0 credits) — location_search, industry_search, etc.

3. Run count-first POST /search-company page 1 requests (each reads total_count — I only pay 1 credit if I actually pull page 1 results, and even that is 1 credit; so keep it to page 1 reads). Produce a sizing table:
   - Baseline: my full multi-signal net → total_count
   - Then vary ONE lever at a time to show segment sizes, e.g.:
     * by headcount band (company_headcount_range: "51-100" vs "101-200" vs "201-500" vs "501-1000")
     * by geo (company_location_search each country separately)
     * with a compliance/AI gate (company_attributes { has_soc2: true } or { uses_ai: true })
     * with a funding gate (company_funding { stage: [...], funding_date: 365 }) — Starter+
   Report each variation's total_count in a markdown table.

4. Recommend which segments are worth a full pull based on size and estimated credit cost (pages = ceil(total_count / 25), 1 credit each). Note that re-pulling the same page within 30 days is free.

Do NOT paginate or enrich anything in this stage.
```

---

### 3. Custom Segmentation — carve the TAM into ICP tiers with high-leverage levers

```
You have the Prospeo MCP + PROSPEO_API_KEY (X-KEY header, host https://api.prospeo.io). I already have a company TAM (either the deduped list from stage 1, or I'll rebuild the net). Segment it into ranked ICP tiers.

MY ICP + TIER LOGIC:
<<< define what makes an account Tier A vs B vs C — e.g. "Tier A = 200–1000 employees AND SOC2 AND venture-backed; Tier B = same size, no SOC2; Tier C = 50–200 employees" — as a security vendor targeting mid-market, my best-fit is Tier A >>>

Do this:

1. Resolve any needed values via /search-suggestions (free).

2. For each tier, build a distinct /search-company filter set layering the discriminating levers on top of the base net. Draw from Prospeo's high-leverage company filters:
   - company_headcount_range / company_headcount_custom  (size bands)
   - company_revenue { min, max, include_unknown_revenue }  (Starter+)
   - company_funding { stage, funding_date, investors, was_in_accelerator }  (Starter+)
   - company_attributes { has_soc2, has_iso27001, has_gdpr, uses_ai, is_venture_backed, has_api, ... }  (Free)
   - company_type { status, subtypes, business_model, is_mainly_ai, has_public_pricing }  (Free)
   - company_technology { include, exclude }  (Starter+, max 20 each, resolve via suggestions)
   - company_headcount_by_department [{ department, min, max }]  (Free)

3. COUNT-FIRST each tier (read total_count) and print a tier-sizing table BEFORE pulling anything.

4. Pull each tier (paginate page 1 → total_page, cap 1000), tag every company with its tier, dedupe by domain across tiers (a company qualifies for the highest tier only), and save one segmented list locally.

5. RE-CLASSIFY from the description, not the tag: for each kept company use the returned description_seo / description_ai / keywords[] to confirm real ICP fit — company_industry tags undercount and mis-tag. Flag any company whose description doesn't match its assigned tier.
```

---

### 4. Signal Layering — add timing/intent signals to prioritize the segmented accounts

```
You have the Prospeo MCP + PROSPEO_API_KEY (X-KEY header, host https://api.prospeo.io). Take my segmented account list and layer in Prospeo's timing/intent signals to rank who to hit NOW.

MY ICP + WHAT SIGNALS MATTER:
<<< which buying signals map to my motion — e.g. "a fintech selling to engineering teams cares about: hiring engineers, recent funding, new CTO, and growing eng headcount" >>>

Do this:

1. Resolve any values via /search-suggestions (free) — e.g. company_google_discovery_search for SEO keywords, company_funding_investors_search, job_title_search.

2. Run count-first /search-company passes that add ONE signal at a time to my qualified net, so I can see how many accounts carry each signal. Choose from Prospeo's signal filters (note plan gates):
   - company_headcount_growth { timeframe_month, min, max, departments }  (Free)
   - company_job_posting_hiring_for { boolean_search: "(engineer OR developer) AND senior AND !intern" }  (Starter+)
   - company_job_posting_quantity { min, max }  (Starter+)
   - company_funding { funding_date: 365, stage: [...] }  (Starter+)
   - company_news { keywords, categories: ["Funding & Investment","Leadership Change","Expansion","Mergers & Acquisitions"], timeframe_days }  (Free)
   - company_key_execs { event_types: ["CTO Appointed","Any C-Level Departed"], timeframe_days }  (Pro+)
   - company_intent { topic_ids: ["<dashboard topic id>"], active_research: true, in_depth_research: true }  (topic_ids are configured in the Prospeo dashboard, not enumerable via API)
   - company_website_traffic { min_monthly_visits, visit_change }  (Pro+)
   If a Pro-gated filter returns PLAN_REQUIRED, skip it and note which key/plan would unlock it.

3. Pull each signal pass, then score every account: base ICP tier + a point per signal present. Output a ranked account list (highest signal-count first) with a column per signal showing which fired. Save locally and report how many accounts carry 2+ signals.
```

---

### 5. People Finder — find persona buyers at the prioritized accounts (contactable only)

```
You have the Prospeo MCP + PROSPEO_API_KEY (X-KEY header, host https://api.prospeo.io). I have a ranked list of qualified account domains. Find the right buyers at each, filtered to contactable people.

MY PERSONA:
<<< target titles / seniority / department — e.g. "CISO, Head of Security, VP Security, or CTO; C-Suite / VP / Head / Director; in Security or IT" >>>

Do this:

1. Resolve titles via /search-suggestions job_title_search (free) if you want exact strings; boolean_search accepts free title terms directly.

2. Feed my qualified domains back through the API-ONLY company.websites.include pre-filter (max 500 domains per call — batch if I have more) on POST /search-person, combined with person filters:
   - company: { websites: { include: [ ...≤500 domains... ] } }
   - person_job_title: { boolean_search: "<<< (CISO OR \"Head of Security\" OR \"VP Security\" OR CTO) AND !Intern >>>" }
   - person_seniority: { include: [ <<< "C-Suite","Vice President","Head","Director" >>> ] }
   - person_department: { include: [ <<< "Information Technology" >>> ] }   (optional, tightens function)
   - person_contact_details: { email: ["VERIFIED"], operator: "OR", hide_people_with_details_already_revealed: false }
   - max_person_per_company: <<< e.g. 3 >>>   (spread contacts evenly across accounts)

3. COUNT-FIRST (read pagination.total_count) and report the persona pull size + estimated search-credit cost (pages × 1) BEFORE pulling. GET /account-information to confirm budget.

4. Paginate page 1 → total_page (cap 1000). Search returns MASKED email/mobile plus a person_id — collect person_id + name + title + company for every result. Do NOT enrich yet (that's stage 6).

5. Dedupe people, keep person_id, and save the list locally with account tier/signal-rank carried over from the prior stages.
```

---

### 6. Enrichment + Clean Handoff — bulk-enrich verified emails and export an outreach-ready file

```
You have the Prospeo MCP + PROSPEO_API_KEY (X-KEY header, host https://api.prospeo.io). I have a people list with person_id from the People Finder stage. Enrich verified emails and produce a clean handoff file.

CONTACT REQUIREMENTS:
<<< which contact fields you need — e.g. "verified work email required; mobile only for Tier A accounts" >>>

Do this:

1. PRE-FLIGHT BUDGET. GET /account-information (free) and report remaining_credits + next_quota_renewal_days. Enrich-person costs 1 credit per email found; enrich_mobile costs 10 credits per mobile. NO_MATCH is never charged; re-enriching the same record within 90 days is free (free_enrichment:true).

2. BULK ENRICH via raw REST POST /bulk-enrich-person — up to 50 records per request (preferred over 50 single calls). Match on person_id from the search results. Set only_verified_email: true so credits are spent only on verified-email hits. Add enrich_mobile: true ONLY for the accounts my requirements flag as mobile-worthy (10 credits each). Loop through my people in batches of 50.

3. Track credit spend as you go from the response, and note any free_enrichment:true rows (90-day dedup, no charge).

4. CLEAN HANDOFF. Produce a single deduped export (CSV) with: full_name, job_title, verified_email, mobile (where pulled), company_name, domain, account_tier, signal_rank, person_id. Drop rows with no verified email. Report final counts: people enriched, verified emails found, mobiles found, total credits spent, and how many were free via the 90-day dedup window.
```

---

*Every Prospeo detail above (filter param names, endpoints, credit costs, plan gates, dedup windows, caps) is grounded in the internal Prospeo TAM Build Guide. Filters marked Starter+/Growth+/Pro+ require that plan — a filter above your plan returns HTTP 400 `PLAN_REQUIRED`.*
