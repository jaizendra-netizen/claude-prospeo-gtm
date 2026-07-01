# Prospeo Filter Reference (Claude Code + Prospeo)

> **Scope:** How to build a highly specific B2B TAM using the Prospeo **API** (NOT the web UI).
> Merged and deduped from Prospeo's own docs: `filters-documentation.md`, `PROSPEO_FILTERS_CHEATSHEET.md`, `search-company.md`, `company-object.md`, `search-person.md`, `person-object.md`, `search-suggestions.md`, `enrich-person.md`, `enrich-company.md`, `account-information.md`, `rate-limits.md`, and the enum pages (departments, industries, seniorities).
> Everything here is grounded in those docs. Nothing is invented.
>
> **Host:** `https://api.prospeo.io` · **Auth header:** `X-KEY: <api_key>` (NOT `Authorization: Bearer`) · **Content-Type:** `application/json`

---


## 1. Why API/CLI beats the UI for TAM

Concrete, doc-supported reasons:

1. **Count-first, free TAM sizing.** Every `/search-company` and `/search-person` response carries a `pagination` object `{current_page, per_page:25, total_page, total_count}`. `total_count` is the exact size of the filtered set — you read it from **page 1** and know the TAM before pulling the rest. `GET /account-information` (free, no params) returns `remaining_credits`, `used_credits`, `next_quota_renewal_days` so you can budget the pull before spending.
2. **Precise credit cost.** Search = **1 credit per page that returns ≥1 result** (25 results/page ⇒ 1 credit buys up to 25 companies or people). `/search-suggestions` = **0 credits**. `/account-information` = **free**. Enrich-person = **1 credit per email found**; with `enrich_mobile` = **10 credits per mobile found** (email included free); enrich-company = **1 credit per matched company**. No charge on `NO_MATCH`.
3. **30-day search dedup (free repeats).** If your team already paid for the *exact same result page* within **30 days**, the identical request is FREE — response returns `"free": true`. Re-paginating a build you already ran costs nothing inside the window.
4. **90-day enrich dedup.** Re-enriching the *same record* within **90 days** is free; response returns `free_enrichment: true`. Exception: re-enriching with `enrich_mobile` when you didn't before → `free_enrichment:false` (you pay for the mobile).
5. **Hard caps you can plan around.** 25 results/page (FIXED); max page = **1,000**; therefore **max 25,000 results per search**. Max 20,000 total filter values across all filters; max 500 include/exclude items per filter (unless a filter overrides). To exceed 25k, split the TAM into non-overlapping sub-searches (by geo, size band, etc.) and union + dedupe.
6. **Boolean / array combinations the UI hides.** `person_job_title` and `company_job_posting_hiring_for` accept a `boolean_search` string (`(CEO OR CTO) AND !Intern`, `"exact"`, `'two words'`, `!exclude`, `()` grouping to depth 5) — far richer than clicking chips.
7. **The full filter surface (40+ filters).** The docs state "more than 20 filters"; the merged taxonomy below is 40+, including levers most UIs never expose.
8. **API-only filters.** `company.names` / `company.websites` (pre-filter against a list of up to 500 names/domains) exist **only** via the API — not in the dashboard builder. Great for seeding a TAM from an existing account list.
9. **High-leverage signals in one call:** lookalike (`company_lookalike`, 4 seed modes), buying-intent (`company_intent`), technographics (`company_technology`, 4,946 values), Google-ranking discovery (`company_google_discovery`), web-traffic (`company_website_traffic`), exec-change triggers (`company_key_execs`), ICP-of-the-company (`company_icp`).
10. **Bulk enrich.** `/bulk-enrich-person` and `/bulk-enrich-company` enrich **up to 50 records per request** — preferred over 50 single calls.

---

## 2. How to drive it (3 ways)

### (a) Raw REST via curl

Base host `https://api.prospeo.io`. Every endpoint uses `X-KEY` + `Content-Type: application/json`. Request body is always `{ "filters": { ... }, "page": N }` (`page` optional, defaults 1).

```bash
curl --location 'https://api.prospeo.io/search-company' \
  --header 'X-KEY: <api_key>' \
  --header 'Content-Type: application/json' \
  --data '{
    "page": 1,
    "filters": {
      "company_industry": { "exclude": ["Semiconductors", "Pet Services"] },
      "company_funding": {
        "stage": ["Series B", "Series C"],
        "funding_date": 365,
        "last_funding":  { "min": "1M",  "max": "10M" },
        "total_funding": { "min": "10M", "max": "100M" }
      }
    }
  }'
```

**Success response:**
```json
{ "error": false, "free": false,
  "results": [ { "company": { ... } } ],
  "pagination": { "current_page": 1, "per_page": 25, "total_page": 11, "total_count": 271 } }
```
**Failure:** `{ "error": true, "error_code": "INVALID_FILTERS", "filter_error": "Invalid industry(s) in include list: ['Accountingg']" }`

Endpoints: `POST /search-company`, `POST /search-person`, `POST /search-suggestions`, `POST /enrich-person`, `POST /enrich-company`, `POST /bulk-enrich-person`, `POST /bulk-enrich-company`, `GET /account-information`.

### (b) The pre-configured MCP servers (`prospeo` / `prospeo-hosted`)

Claude Code already has the `prospeo` / `prospeo-hosted` MCP servers configured. **They expose only `enrich` + `search` + `account`** — they do **NOT** expose `/search-suggestions` or the `/bulk-*` endpoints. Consequences:

- Use the MCP for quick single searches, single enriches, and credit checks.
- **Big TAM builds go through raw REST** (section a) because you need: `/search-suggestions` to resolve non-enumerable values (locations, titles, technologies, investors, ICP fields, etc.) and `/bulk-enrich-*` to enrich 50-at-a-time. Neither is available on the MCP.

### (c) Scripted pagination loops

Because a single search caps at 25,000 results (1,000 pages × 25) and results come 25/page, non-trivial builds require a loop:

```bash
#!/usr/bin/env bash
KEY="$PROSPEO_API_KEY"; FILTERS='{ ... your filters ... }'
# 1) count-first
resp=$(curl -s https://api.prospeo.io/search-company -H "X-KEY: $KEY" \
  -H 'Content-Type: application/json' --data "{\"page\":1,\"filters\":$FILTERS}")
total_page=$(echo "$resp" | jq '.pagination.total_page')
echo "TAM total_count=$(echo "$resp" | jq '.pagination.total_count'), pages=$total_page"
# 2) paginate 1..min(total_page,1000), collecting results[].company, dedupe by domain
for p in $(seq 1 $(( total_page < 1000 ? total_page : 1000 ))); do
  curl -s https://api.prospeo.io/search-company -H "X-KEY: $KEY" \
    -H 'Content-Type: application/json' --data "{\"page\":$p,\"filters\":$FILTERS}" \
    | jq -c '.results[].company' >> companies.ndjson
  # throttle: read x-minute-request-left / x-daily-request-left headers; back off on 429
done
sort -u -t'"' -k? companies.ndjson  # dedupe by the .domain field
```

**Throttle from live headers** on every response: `x-minute-request-left`, `x-daily-request-left`, `x-minute-reset-seconds`, `x-daily-reset-seconds` (plus `x-*-rate-limit`). A `429 "Rate limit exceeded"` means back off and wait `x-minute-reset-seconds`.

**Search rate limits** (per/sec · /min · /day): Starter `1·30·1,000` · Growth `2·60·4,000` · Pro `5·180·250,000`.
**Enrich rate limits** (separate bucket): Starter `5·300·2,000` · Growth `5·300·5,000` · Pro `30·1,800·500,000`.

---

## 3. The COMPLETE filter taxonomy

Legend: ⭐ = high-leverage / plan-gated. Plan tags: `(Free)` `(Starter+)` `(Growth+)` `(Pro+)`. Company filters work on **both** `/search-company` and `/search-person`; person filters work only on `/search-person`.

> **Global rules:** at least ONE positive (include) filter is required — exclude-only searches are rejected. Include/exclude arrays default max 500 items unless a filter overrides (many override to 20, 10, 50, 100, or 11). Values for non-enumerable filters must be resolved verbatim via `/search-suggestions` — arbitrary strings like `"nyc"` are rejected.

### 3A. Company — Firmographic / company-type

| Param | What it filters | Value type | Example |
|---|---|---|---|
| ⭐ `company` (API-only) | Pre-filter to a list of company names / root domains | object `{names:{include,exclude}, websites:{include,exclude}}` | `{"websites":{"include":["deloitte.com"]},"names":{"include":["Walmart","Microsoft"]}}` — max 500 total; root domains only |
| ⭐ `company_type` (Free) | Status, subtypes, business model + 12 GTM flags | object | `{"status":"Private","subtypes":{"include":["SaaS","Platform"]},"business_model":"b2b","is_mainly_ai":true,"has_public_pricing":true}` |
| `company_founded` (Free) | Founding-year range | `{min,max,include_unknown_founded}` | `{"min":2010,"max":2020,"include_unknown_founded":true}` (1900–current) |
| ⭐ `company_awards` (Growth+) | Awards / recognitions | `{include (max 50), match_mode:EXACT\|CONTAINS}` | `{"include":["Inc 5000","Forbes Cloud 100"],"match_mode":"EXACT"}` (via suggestions) |
| ⭐ `company_key_customers` (Growth+) | Vendors whose customers include named brand X | `{include}` (include-only, max 100) | `{"include":["Google","Microsoft","Amazon"]}` (via suggestions) |
| `company_operating_languages` (Free) | Languages the company operates in | `{include}` (max 10) | `{"include":["English","French","German"]}` (via suggestions) |

`company_type` fields: `status` (Private/Public/Non Profit/Other); `subtypes.include/exclude` (SaaS, Marketplace, E-commerce, Agency, Consulting, Manufacturing, Media/Publisher, Education, Non-Profit, Government, FinTech, HealthTech, AI/ML, Hardware, Professional Services, Platform, Data/Analytics, Franchise, Logistics, Real Estate, Legal, Insurance, Retail, Hospitality, Food & Beverage, Construction, Telecommunications); `business_model` (b2b, b2c, b2b2c, d2c, marketplace, franchise, non_profit, government); boolean-or-null flags: `is_retail, is_marketplace, ⭐is_mainly_ai, is_mainly_crypto, multi_product, has_free_tier, is_self_serve, is_sales_led, has_usage_pricing, has_subscription, has_enterprise_plan, has_public_pricing`.

### 3B. Company — Geo / location

| Param | What it filters | Value type | Example |
|---|---|---|---|
| `company_location_search` (Free) | HQ location | `{include,exclude}` (max 100; via suggestions) | `{"include":["United States","San Francisco, California, United States"],"exclude":["China"]}` |
| ⭐ `company_headcount_by_location` (Growth+) | Employee count within specific countries | `{entries:[{country,min_headcount,max_headcount}]}` (1–10 entries) | `{"entries":[{"country":"United States","min_headcount":50,"max_headcount":500}]}` |

### 3C. Company — Industry classification

| Param | What it filters | Value type | Example |
|---|---|---|---|
| `company_industry` (Free) | Main industry (256-value enum) | `{include,exclude}` (max 500 each) | `{"include":["Software Development","IT Services and IT Consulting"],"exclude":["Construction"]}` |
| `company_naics` (Free) | NAICS codes | `{include,exclude}` int codes (max 100 each) | `{"include":[541511,541512],"exclude":[236220]}` |
| `company_sics` (Free) | SIC codes | `{include,exclude}` int codes (max 100 each) | `{"include":[7371,7372],"exclude":[1521]}` |

> **⚠ Never seed a TAM on `company_industry` alone** — it undercounts (a security vendor tagged "Software Development", another security vendor tagged "Computer & Network Security"). Use a multi-signal net and re-classify from the returned `description_seo`/`description_ai`.

### 3D. Company — Size / headcount

| Param | What it filters | Value type | Example |
|---|---|---|---|
| `company_headcount_range` (Free) | Predefined employee bands | array of enum | `["51-100","101-200","201-500"]` |
| `company_headcount_custom` (Free) | Custom employee range | `{min,max}` — `min` 1–999,998, `max` 1–999,999 (must be ≥ min) | `{"min":100,"max":5000}` — mutually exclusive with `company_headcount_range` |
| ⭐ `company_headcount_by_department` (Free) | Headcount inside specific departments | array `[{department,min,max}]` (max 10) | `[{"department":"Sales","min":10,"max":100},{"department":"Engineering & Technical","min":50,"max":null}]` |

### 3E. Company — Revenue

| Param | What it filters | Value type | Example |
|---|---|---|---|
| ⭐ `company_revenue` (Starter+) | Revenue band | `{min,max,include_unknown_revenue}` (Revenue enum strings) | `{"min":"10M","max":"500M","include_unknown_revenue":false}` |

### 3F. Company — Funding

| Param | What it filters | Value type | Example |
|---|---|---|---|
| ⭐ `company_funding` (Starter+) | Stage, recency, amounts, investors, accelerator | object | `{"stage":["Series A","Series B"],"funding_date":365,"last_funding":{"min":"1M","max":"100M"},"total_funding":{"min":"5M","max":"500M"},"investors":["Sequoia Capital"],"was_in_accelerator":true,"accelerator_name":"Y Combinator"}` |

`stage` = Funding Stage enum (max 500); `funding_date` = days since last funding; `last_funding`/`total_funding` use Revenue-range enum strings; `investors` max 10; `accelerator_name` used only when `was_in_accelerator:true`. Investors & accelerators resolved via suggestions.

### 3G. Company — Technographic

| Param | What it filters | Value type | Example |
|---|---|---|---|
| ⭐ `company_technology` (Starter+) | Tech stack (4,946-value enum) | `{include,exclude}` (max 20 each) | `{"include":["Salesforce","HubSpot","AWS"],"exclude":["Azure"]}` |
| ⭐ `company_attributes` (Free) | ~45 boolean product/compliance/presence flags | object | `{"b2b":true,"uses_ai":true,"has_api":true,"has_soc2":true,"has_gdpr":true,"has_case_studies":true}` |
| ⭐ `company_integrations` (Growth+) | Integrations named on the website | `{include (max 20),exclude (max 10)}` | `{"include":["Salesforce","HubSpot","Slack"],"exclude":["Zendesk"]}` (via suggestions) |
| `company_email_provider` (Free) | MX / email provider | array | `["Google","Microsoft"]` — filterable: Microsoft, Google, Proofpoint, Mimecast, Other |

`company_attributes` groups (each field true=require / false=exclude / omit=ignore): **General** b2b, demo, freetrial, downloadable, mobileapps, onlinereviews, pricing, `uses_ai`; **Platform** has_api, has_chrome_extension, has_sso, has_uptime_guarantee, has_open_source, has_marketplace; **Content** has_blog, has_podcast, has_community_forum, has_knowledge_base, has_academy, has_affiliate_program, has_case_studies, has_testimonials; **Support** has_phone_support, has_email_support, has_chat_support, has_ticket_support, has_social_support; **Compliance** has_soc2, has_iso27001, has_gdpr, has_hipaa, has_ccpa, has_pci_dss, `other_compliance` (array max 50), `compliance_match_mode` (EXACT/CONTAINS), has_esg_reports; **Presence** data_residency (string), has_physical_offices, is_venture_backed, is_publicly_traded.

### 3H. Company — AI-native signals

| Param | What it filters | Value type | Example |
|---|---|---|---|
| ⭐ `company_type.is_mainly_ai` (Free) | AI-native company flag | boolean | `{"is_mainly_ai":true}` |
| ⭐ `company_attributes.uses_ai` (Free) | Company uses AI | boolean | `{"uses_ai":true}` |
| ⭐ `company_key_execs` (Pro+) | Recent exec appointments/departures | `{event_types,timeframe_days}` | `{"event_types":["CTO Appointed","Any C-Level Departed"],"timeframe_days":180}` |

`company_key_execs` event types: `{CEO,CTO,CFO,COO,CMO,CRO} {Departed,Appointed}`, `VP of {Sales,Marketing,Engineering} {Departed,Appointed}`, wildcards `Any {C-Level,VP,Director} {Departed,Appointed}` (1–10 items; `timeframe_days` default 90). *(Note: also useful as an AI-native tell alongside the returned `description_ai` field.)*

### 3I. Company — Intent

| Param | What it filters | Value type | Example |
|---|---|---|---|
| ⭐ `company_intent` (Free*) | Buying-intent stage on your configured topics | `{topic_ids,early_research,active_research,in_depth_research}` | `{"topic_ids":["<topic_id>"],"active_research":true,"in_depth_research":true}` |
| ⭐ `company_news` (Free) | Recent news by keyword / category | `{keywords,categories,timeframe_days}` | `{"keywords":["acquisition"],"categories":["Mergers & Acquisitions","Expansion"],"timeframe_days":180}` |

`company_intent`: `topic_ids` 1–30, each configured in the Prospeo **dashboard** Intent settings — **NOT enumerable via API**. `company_news.categories`: Funding & Investment, Mergers & Acquisitions, Product Launch, Partnership, Expansion, Layoffs & Restructuring, IPO, Leadership Change, Legal & Regulatory, Awards & Recognition.

### 3J. Company — Web-traffic

| Param | What it filters | Value type | Example |
|---|---|---|---|
| ⭐ `company_website_traffic` (Pro+) | Traffic volume, trend, audience countries | object | `{"min_monthly_visits":10000,"max_monthly_visits":1000000,"visit_change":{"period":"quarterly","min_change":10,"max_change":200},"top_countries":["United States","United Kingdom"],"min_country_pct":20}` |

`visit_change.period` = monthly (default) / quarterly / yearly; `top_countries` max 5 (via suggestions).

### 3K. Company — Google-discovery

| Param | What it filters | Value type | Example |
|---|---|---|---|
| ⭐ `company_google_discovery` (Pro+) | Companies by the SEO keywords they RANK for | `{seo_keywords}` (1–100) | `{"seo_keywords":["email finder","sales intelligence","lead generation"]}` (via suggestions) |

### 3L. Company — Lookalike / ICP

| Param | What it filters | Value type | Example |
|---|---|---|---|
| ⭐ `company_lookalike` (Starter+) | AI similarity — companies like a seed | object (4 mutually-exclusive modes) | see below |
| ⭐ `company_icp` (Growth+) | The ICP the company **sells to** | object | `{"titles_include":["CTO","VP Engineering"],"company_sizes":["smb","midmarket","enterprise"],"industries":["Software Development"],"geographic_markets":["US","GB","DE"],"geographic_scope":"multi_country","departments":{"include":["Engineering","IT","Security"],"match_mode":"any"}}` |

`company_lookalike` modes (use exactly one): **A** `company_oids` (max 10 Mongo-style seed IDs) · **B** `icp_text` (free text, max 5000 chars) · **C** `person_oids` (max 10, **`/search-person` only**) · **D** `domain`. Plus `minimum_tier` T1 (most similar) / T2 / T3 (broadest, default), `match_all` (true=intersect seeds, default false=any), `same_language` (default false).
`company_icp`: `company_sizes` = micro/smb/midmarket/enterprise/large_enterprise; ICP `departments.include` = Engineering, Sales, Marketing, Finance, HR, Operations, IT, Legal, Customer Success, Procurement, Data, Security, Design, SMB Owners, Consumers. Titles/industries/markets/departments via suggestions.

### 3M. Company — Growth / hiring

| Param | What it filters | Value type | Example |
|---|---|---|---|
| ⭐ `company_headcount_growth` (Free) | Headcount growth %, overall or per dept | `{timeframe_month,min,max,departments}` | `{"timeframe_month":12,"min":10,"max":100,"departments":["Sales","Technical"]}` |
| ⭐ `company_job_posting_hiring_for` (Starter+) | Titles the company is actively hiring for | `{include,exclude,match_type}` **or** `{boolean_search}` | `{"boolean_search":"(engineer OR developer) AND senior AND !intern"}` |
| ⭐ `company_job_posting_quantity` (Starter+) | Number of active job postings | `{min,max}` (0–5,000) | `{"min":10,"max":100}` |

`company_headcount_growth` min/max −100..10,000%; `departments` (max 10) use the Headcount-Growth department enum (Administrative, Consulting, Customer service, Design / UI / UX, Education, Finance, General management, HR, Legal, Marketing, Medical, Operations, Product, Project management, Real estate, Research, Sales, Technical, Trades).

### 3N. Company — Keyword / website search

| Param | What it filters | Value type | Example |
|---|---|---|---|
| ⭐ `company_keywords` (Free) | Keywords across company content sources | `{include,exclude,include_all,search_everywhere,sources}` (max 20) | `{"include":["saas","b2b","enterprise"],"exclude":["consumer"],"include_all":false,"search_everywhere":false,"sources":["specialties","seo_description","website_pages"]}` |
| ⭐ `company_website_search` (Pro+) | Full-text over crawled site pages + page-presence flags | object | `{"include_keywords":["enterprise pricing","SOC 2"],"exclude_keywords":["free trial"],"match_mode":"any","page_scope":["product","about"],"url_contains":"pricing","has_developer_docs_page":true,"has_security_page":true}` |
| `company_products_services` (Free) | Product & service tags | `{products_include/exclude,products_match_all,service_tags_include/exclude,service_tags_match_all}` | `{"products_include":["CRM","email automation"],"products_exclude":["accounting"],"service_tags_include":["consulting"]}` (via suggestions) |

`company_keywords.sources` valid values: specialties, social_media_description, seo_description, ai_description, products_services, website_pages. `include_all:true` = AND logic.
`company_website_search`: `search_in` flags page_body (default true), page_titles, urls_only, headings_only, seo_description; `page_scope` (max 5): homepage, product, blog, careers, about; website-presence booleans: has_persona_pages, has_industry_pages, has_solution_pages, has_careers_page, has_status_page, has_sla_page, has_developer_docs_page, has_investor_page, has_security_page, has_comparison_pages.

### 3O. Company — Name / domain & exclusions

- `company.names` / `company.websites` — see 3A (API-only, max 500).
- Every include/exclude filter supports **negatives** via its `exclude` array (industry, technology, integrations, keywords, etc.). **You cannot run an exclude-only search.**
- **Unavailable via API** (dashboard-only, will not work): `company.company_oids`, `company.company_list_oids`, `company.temp_matching_oids`.

### 3P. Person — Title / role

| Param | What it filters | Value type | Example |
|---|---|---|---|
| ⭐ `person_job_title` | Job title; exact/contains/similar or boolean | `{include,exclude,match_mode,smart_intensity}` **or** `{boolean_search}` | `{"boolean_search":"(CEO OR CTO OR \"Chief Technology Officer\") AND !Intern"}` |
| `person_search` | Name OR company name/domain match | `{include,exclude,match_mode:SMART\|STRICT}` | `{"include":["John Smith"],"exclude":["Bob Wilson"],"match_mode":"SMART"}` |
| `person_name` | Strict name-only | `{include,exclude}` | `{"include":["John Smith","Jane Doe"]}` — don't send with `person_search` |
| `person_name_or_job_title` | Free-text across name + title | string | `"Kevin"` |
| `person_year_of_experience` | Total years of experience | `{min,max}` (0–60) | `{"min":5,"max":15}` |
| ⭐ `person_time_in_current_role` | Months in current role | `{min,max}` (0–600) | `{"min":6,"max":24}` |
| ⭐ `person_time_in_current_company` | Months at current company | `{min,max}` (0–600) | `{"min":12,"max":60}` |

`person_job_title` `match_mode` = CONTAINS / EXACT / SIMILAR (SIMILAR expands CEO→Chief Executive Officer; `smart_intensity` LOOSE/NORMAL/STRICT applies only with SIMILAR). `boolean_search` max 500 terms, paren depth 5, cannot combine with include/exclude. Titles resolved via `/search-suggestions job_title_search`.

### 3Q. Person — Seniority / department / skills

| Param | What it filters | Value type | Example |
|---|---|---|---|
| `person_seniority` | Seniority level (10-value enum) | `{include,exclude}` (max 11) | `{"include":["C-Suite","Vice President","Director"]}` |
| `person_department` | Functional department (14 mains + subs) | `{include,exclude}` | `{"include":["Sales","Marketing"]}` |

> **Skills:** there is **no `person_skills` filter** in the docs. `skills[]` is a **returned field** on the person object (usable post-pull, not as a search filter).

### 3R. Person — Location & contact-availability

| Param | What it filters | Value type | Example |
|---|---|---|---|
| `person_location_search` | Person location | `{include,exclude}` (max 100; via suggestions) | `{"include":["United States","New York, United States"],"exclude":["Texas, United States"]}` |
| ⭐ `person_contact_details` | Availability of verified email / mobile | `{email,mobile,operator,hide_people_with_details_already_revealed}` | `{"email":["VERIFIED"],"mobile":["VERIFIED"],"operator":"OR","hide_people_with_details_already_revealed":false}` |
| ⭐ `max_person_per_company` | Cap people returned per company | int (1–100) | `5` |

`person_contact_details`: `email` valid `["VERIFIED"]`; `mobile` valid `["VERIFIED","UNAVAILABLE"]`; `operator` OR (default) / AND.

### 3S. Person — Signals & company-context

| Param | What it filters | Value type | Example |
|---|---|---|---|
| ⭐ `person_job_change` (Growth+) | Recent promotions / company changes | `{timeframe_days,only_promotion,only_new_company}` | `{"timeframe_days":90,"only_promotion":false,"only_new_company":true}` |

**Company-context on person search:** every company filter in 3A–3O may be added to a `/search-person` request to constrain *which companies* the people work at (e.g. find CTOs *at* Series-B fintechs). Example combined body:
```json
{"filters":{
  "person_job_title":{"boolean_search":"(CTO OR \"VP Engineering\")"},
  "person_seniority":{"include":["C-Suite","Vice President"]},
  "company_industry":{"include":["Software Development"]},
  "company_headcount_range":["101-200","201-500"],
  "company_funding":{"stage":["Series B"],"funding_date":365}
},"page":1}
```

---

## 4. Valid enum values

### Seniorities (10 — full enum)
`Founder/Owner`, `C-Suite`, `Partner`, `Vice President`, `Head`, `Director`, `Manager`, `Senior`, `Entry`, `Intern`.

### Departments — 14 main (submitting a main auto-includes its subs)
`C-Suite`, `Consulting`, `Design`, `Education & Coaching`, `Engineering & Technical`, `Finance`, `Human Resources`, `Information Technology`, `Legal`, `Marketing`, `Medical & Health`, `Operations`, `Product`, `Sales`.

Representative sub-departments: **C-Suite** → Chief Executive, Finance Executive, Founder, HR Executive, IT Executive, Legal Executive, Marketing Executive, Medical & Health Executive, Operations Executive, Sales Leader. **Engineering & Technical** → AI/ML, Data Science & Analysis, DevOps, Mobile Development, Software Development, Test/QA, Research & Development, Scrum Master/Agile Coach, Business Intelligence, Digital Transformation. **Information Technology** → Cloud Engineering, Information Security, Networking, Database Administration, IT Strategy, Help Desk/Desktop Services, Technical Lead. **Sales** → Account Executive, Sales Development, Revenue Operations, Sales Engineering, Customer Success, Partnerships, Account Management, Channel Sales. **Marketing** → Growth & Demand Generation, Product Marketing, Digital Marketing, Content Marketing, SEO, Brand Management, Marketing Operations. (Finance, HR, Legal, Medical & Health, Operations, Product, Design, Consulting, Education & Coaching also have full sub-trees — see `enum_departments.md`.)

> **Separate Headcount-Growth department enum** (only for `company_headcount_growth.departments`): Medical, Sales, HR, Legal, Marketing, Finance, Technical, Consulting, Operations, Product, General management, Administrative, Customer service, Project management, Design / UI / UX, Research, Trades, Real estate, Education.

### Revenue-range enum (used by `company_revenue`, and `company_funding.last_funding`/`total_funding`)
`<100K`, `100K`, `500K`, `1M`, `5M`, `10M`, `25M`, `50M`, `100M`, `250M`, `500M`, `1B`, `5B`, `10B+`.

### Headcount-range enum
`1-10`, `11-20`, `21-50`, `51-100`, `101-200`, `201-500`, `501-1000`, `1001-2000`, `2001-5000`, `5001-10000`, `10000+`.

### Funding-stage enum (exact strings — Prospeo rejects non-verbatim values)
`Angel`, `Convertible note`, `Corporate round`, `Debt financing`, `Equity crowdfunding`, `Grant`, `Initial coin offering` (NOT "ICO"), `Non equity assistance`, `Post IPO debt`, `Post IPO equity`, `Post IPO secondary` (three separate values — NOT "Post IPO debt/equity/secondary"), `Pre seed`, `Private equity`, `Product crowdfunding`, `Secondary market`, `Seed`, `Series A`, `Series B`, `Series C`, `Series D` (list individually — "Series A-D" is NOT valid), `Series E-J`, `Series unknown`, `Undisclosed`.

### Industries (256 total — representative slice)
IT Services and IT Consulting; Software Development; Business Consulting and Services; Financial Services; Technology, Information and Internet; Computer and Network Security; Data Infrastructure and Analytics; Data Security Software Products; Advertising Services; Marketing Services; Staffing and Recruiting; Human Resources Services; Insurance; Banking; Capital Markets; Venture Capital and Private Equity Principals; Investment Management; Hospitals and Health Care; Medical Practices; Biotechnology Research; Pharmaceutical Manufacturing; Medical Equipment Manufacturing; Construction; Real Estate; Manufacturing (General); Semiconductor Manufacturing; Semiconductors; Telecommunications; Wireless Services; Renewable Energy; Oil, Gas, and Mining; Higher Education; E-Learning Providers; Retail (General / Apparel and Fashion / Groceries / Health and Personal Care); Internet Marketplace Platforms; Social Networking Platforms; Blockchain Services; Computer Games; Mobile Gaming Apps; Robotics Engineering; Nanotechnology Research; Defense and Space Manufacturing; Aviation and Aerospace Component Manufacturing; Transportation, Logistics, Supply Chain and Storage; Law Practice / Legal Services; Accounting; Government Administration; Non-profit Organizations. *(Resolve any specific value with `/search-suggestions industry_search`; full list in `enum_industries.md`.)*

### Big enum pages (not inlined — resolve as needed)
Industries **256**, Technologies **4,946**, NAICS codes, SIC codes, MX providers (**5** filterable / 105 in response). Technologies and locations are best resolved live via `/search-suggestions`.

---

## 5. A concrete high-quality TAM build recipe

**Goal example:** cybersecurity vendors in US / UK / Canada, then their security/exec buyers, enriched with verified emails.

**Credit math up front:** search = 1 credit / 25 results. A 2,000-company TAM ⇒ 80 search credits. Persona pull of 5,000 people ⇒ 200 search credits. Enrich 5,000 emails ⇒ 5,000 credits (1/email); add mobiles ⇒ +10/mobile. Repeats within the dedup windows are free.

**Step 1 — Scope.** Nail ICP (industry/sub-niche, size, revenue, geo, funding, public/private), persona (titles/seniority/department), signals to layer (hiring/funding/news/tech/intent/traffic/lookalike), and the definition edge.

**Step 2 — Resolve values via `/search-suggestions`** (FREE, 0 credits, 15 req/s all plans, exactly ONE `*_search` param per call, ≥2 chars). Translate human phrases into canonical stored strings/codes, used verbatim:
- `location_search` → `"United States"`, `"United Kingdom"`, `"Canada"`
- `industry_search "security"` → `"Computer and Network Security"`, etc.
- `technology_search`, `company_funding_investors_search`, `company_key_customers_search`, `company_icp_titles_search`, `job_title_search`, `company_google_discovery_search`, `company_awards_compliance_search "soc 2"` …

**Step 3 — Count-first to size the TAM.** POST `/search-company` page 1 with a **multi-signal** net and read `pagination.total_count` before pulling:
```json
{ "page": 1, "filters": {
  "company_location_search": { "include": ["United States","United Kingdom","Canada"] },
  "company_industry": { "include": ["Computer and Network Security","Software Development","IT Services and IT Consulting"] },
  "company_keywords": {
    "include": ["cybersecurity","security","threat detection","SIEM","SOC","appsec","vulnerability","zero trust","endpoint","SASE"],
    "search_everywhere": false,
    "sources": ["specialties","seo_description","ai_description","website_pages"]
  }
}}
```

**Step 4 — Build the filtered `/search-company`** and tighten/segment with high-leverage levers: `company_attributes.has_soc2`/`uses_ai`, `company_type.is_mainly_ai`, `company_funding.stage`, `company_headcount_range`, `company_website_search.has_security_page:true` (Pro), `company_google_discovery.seo_keywords:["cybersecurity software"]` (Pro). If `total_count > 25,000`, split into non-overlapping sub-searches (by geo or size band) and union.

**Step 5 — Paginate to the cap.** Loop `page` 1 → `total_page` (max 1,000). Collect `results[].company`. Throttle from `x-minute-request-left` / `x-daily-request-left`; back off on 429.

**Step 6 — Dedupe by domain.** The company object returns `domain` (root, e.g. `intercom.io`) — dedupe on that, not on name/website.

**Step 7 — Re-classify from the description.** For each company use the returned `description_seo` / `description_ai` (and `keywords[]`), **not** the `company_industry` tag, to confirm ICP fit — industry tags undercount and mis-tag.

**Step 8 — `/search-person` for personas.** Feed the qualified domains back via the API-only `company.websites.include` (max 500 per call, so batch) plus person filters:
```json
{ "page": 1, "filters": {
  "company": { "websites": { "include": ["<qualified domains, ≤500>"] } },
  "person_job_title": { "boolean_search": "(CISO OR \"Head of Security\" OR \"VP Security\" OR CTO)" },
  "person_seniority": { "include": ["C-Suite","Vice President","Head","Director"] },
  "person_contact_details": { "email": ["VERIFIED"], "operator": "OR" },
  "max_person_per_company": 3
}}
```
Search returns masked email/mobile — you get `person_id` to enrich.

**Step 9 — Bulk enrich.** `/bulk-enrich-person` up to **50 records/request**, matching on `person_id` (from search). Set `only_verified_email:true` to only spend credits on verified-email hits (NO_MATCH is not charged). Add `enrich_mobile:true` only where a mobile is essential (10 credits/mobile). Response `free_enrichment:true` = you weren't charged (90-day dedup). Pre-flight budget with `GET /account-information` (`remaining_credits`, `next_quota_renewal_days`).

---

## 6. The high-leverage filters most people miss

- ⭐ **`company` (names/websites)** — API-ONLY list pre-filter (max 500). Seed a TAM directly from an existing account list.
- ⭐ **`company_lookalike`** — 4 seed modes (company OIDs / free-text ICP / person OIDs / a single domain) with T1–T3 tiers.
- ⭐ **`company_icp`** — filter by the ICP the company **sells to** (their target titles/sizes/industries/markets/departments), not their own firmographics.
- ⭐ **`company_google_discovery`** (Pro) — find companies by the SEO keywords they **rank for** on Google.
- ⭐ **`company_website_search`** (Pro) — full-text over crawled pages + presence booleans (`has_security_page`, `has_developer_docs_page`, `has_sla_page`, `has_comparison_pages`, `url_contains`).
- ⭐ **`company_website_traffic`** (Pro) — traffic volume + growth trend + top-country mix.
- ⭐ **`company_key_execs`** (Pro) — CEO/CTO/CFO/CMO/CRO/COO + VP appointments & departures as timed triggers.
- ⭐ **`company_intent`** — buying-intent stages (early/active/in-depth) on dashboard-configured topics.
- ⭐ **`company_key_customers`** (Growth) — find vendors whose customers include a named brand.
- ⭐ **`company_headcount_growth`** & **`company_headcount_by_department`** — pinpoint teams scaling (e.g. Sales +30% in 12 mo).
- ⭐ **`company_job_posting_hiring_for`** (Starter) — boolean-search live job postings as a hiring-intent signal.
- ⭐ **`company_attributes`** — ~45 booleans (`uses_ai`, `has_soc2`, `has_iso27001`, `is_venture_backed`, `has_open_source`, `data_residency`…) for compliance/AI targeting.
- ⭐ **`company_technology`** (Starter) — 4,946-value stack filter with include/exclude.
- ⭐ **`person_job_change`** (Growth) — recent movers/promotions as timing signals.
- ⭐ **`person_contact_details` + `only_verified_email`** — build contactable-only lists and only spend credits on verified matches.
- ⭐ **`max_person_per_company`** — spread contacts evenly across accounts.
- ⭐ **`/search-suggestions`** — FREE, 0 credits: always the first step to resolve locations/titles/technologies/investors/ICP/awards/compliance/products/services verbatim.
- ⭐ **`total_count` count-first + 30-day search dedup + 90-day enrich dedup** — size before spending, and re-runs inside the windows are free.

---

