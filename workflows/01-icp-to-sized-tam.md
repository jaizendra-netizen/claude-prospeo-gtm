# ICP to a sized TAM

**Goal:** Turn an ICP into a precise company TAM using a multi-signal Prospeo filter net, and size it for free with count-first before spending any credits.

## When to use

- You have an ICP (industry, size, geo, tech, attributes, signals) and need the actual set of matching companies, not a guess at market size.
- You want to know how big the TAM is — and how many credits the pull will cost — *before* you paginate.
- You're seeding an outbound list, ABM tier, or persona pull and need clean, deduped, ICP-confirmed accounts.

Do **not** seed a TAM on `company_industry` alone — industry tags undercount and mis-tag (a security vendor can be filed under "Software Development"). Always cast a multi-signal net and re-classify from each company's description.

## Prompt for Claude Code

```
Build me a sized company TAM from this ICP using the Prospeo MCP server + Prospeo API.

ICP:
- Industry / sub-niche: <e.g. cybersecurity vendors>
- Company size: <headcount band or custom min/max>
- Geo (HQ): <e.g. US, UK, Canada>
- Revenue / funding (optional): <band, stage, recency>
- Tech / attributes / signals to layer (optional): <e.g. has_soc2, uses_ai, hiring, intent>
- Persona (for a later people pull): <titles / seniority / department>

Do this, in order:
1. Resolve every human value (locations, industries, technologies, titles) to Prospeo's
   canonical strings via /search-suggestions (0 credits). Never send raw strings like "nyc".
2. Build a MULTI-SIGNAL /search-company net — company_location_search + company_industry +
   company_keywords (+ technology/attributes if relevant). Do NOT rely on industry alone.
3. Count-first: POST page 1 and read pagination.total_count + total_page. Report the TAM size
   and the credit cost of the full pull BEFORE pulling. Wait for my go-ahead.
4. On my go: paginate page 1..total_page (cap 1,000 pages / 25,000 results), collecting
   results[].company.
5. Dedupe by the root `domain` field (not name/website).
6. Re-classify each company from description_seo / description_ai / keywords[], not the
   company_industry tag. Drop anything that doesn't fit the ICP.

Output: a deduped, ICP-confirmed company list (domain + name + description) and the final count
vs. the raw total_count. Set PROSPEO_API_KEY=your_key_here in the environment.
```

## Prospeo filters / endpoints used

Endpoints:
- `POST /search-suggestions` — resolve non-enumerable values verbatim (0 credits). One `*_search` param per call, ≥2 chars: `location_search`, `industry_search`, `technology_search`, `job_title_search`.
- `POST /search-company` — the TAM search. Body is `{ "page": N, "filters": { ... } }`.
- `GET /account-information` — free; returns `remaining_credits`, `used_credits`, `next_quota_renewal_days` to budget before pulling.

Filters (exact param names):
- `company_location_search` — `{include, exclude}` HQ location (max 100; values via suggestions).
- `company_industry` — `{include, exclude}` main industry (256-value enum; max 500 each). Never use alone.
- `company_keywords` — `{include, exclude, include_all, search_everywhere, sources}` (max 20). Valid `sources`: `specialties`, `social_media_description`, `seo_description`, `ai_description`, `products_services`, `website_pages`.
- `company_technology` — `{include, exclude}` (max 20 each; Starter+; 4,946-value enum via suggestions).
- `company_attributes` — ~45 boolean flags, e.g. `uses_ai`, `has_soc2`, `has_gdpr`, `has_api` (Free).
- `company_type` — `{status, subtypes, business_model, is_mainly_ai, ...}` firmographic / GTM flags (Free).
- `company_headcount_range` (array of bands) or `company_headcount_custom` `{min, max}` — mutually exclusive.
- `company_revenue` — `{min, max, include_unknown_revenue}` (Starter+); `company_funding` — `{stage, funding_date, ...}` (Starter+).

Read fields used downstream (returned, not filters): `domain` (root, dedupe key), `description_seo`, `description_ai`, `keywords[]`. Pagination object on every response: `{current_page, per_page: 25, total_page, total_count}`.

## Steps

1. **Scope the ICP.** Nail industry/sub-niche, size, revenue, geo, funding, public/private, plus any signals to layer (tech, attributes, hiring, intent). Decide the definition edge — what's in vs. out.
2. **Resolve values via `/search-suggestions`** (FREE, 0 credits, exactly one `*_search` param per call, ≥2 chars). Translate every human phrase into Prospeo's canonical stored string — e.g. `"security"` → `"Computer and Network Security"`, `"United States"` for location. Raw strings like `"nyc"` are rejected.
3. **Build the multi-signal net.** Combine `company_location_search` + `company_industry` + `company_keywords` (and `company_technology` / `company_attributes` where relevant). Casting across industry **and** keywords **and** attributes catches the companies a single industry tag misses. At least one positive (include) filter is required — exclude-only searches are rejected.
4. **Count-first (free sizing).** POST `/search-company` with `page: 1` and read `pagination.total_count` (exact TAM size) and `total_page` from the response — before pulling any further pages. This spends at most 1 credit (page 1) and tells you the whole TAM. Optionally hit `GET /account-information` to confirm you have the credits.
5. **Segment if over the cap.** A single search caps at 25,000 results (1,000 pages × 25). If `total_count > 25,000`, split into non-overlapping sub-searches (by geo or size band) and union.
6. **Paginate.** Loop `page` from 1 to `min(total_page, 1000)`, collecting `results[].company`. Each page that returns ≥1 result costs 1 credit (25 companies/page). Throttle on the live headers (`x-minute-request-left`, `x-daily-request-left`) and back off on a `429`.
7. **Dedupe by domain.** Dedupe on the returned root `domain` field (e.g. `intercom.io`), not on name or website.
8. **Re-classify from the description.** For each company, confirm ICP fit from `description_seo` / `description_ai` / `keywords[]` — **not** the `company_industry` tag (tags undercount and mis-tag). Drop non-fits. This is your final, sized, ICP-confirmed TAM.

## Credits

`/search-suggestions` and `/account-information` are **free (0 credits)**. Search costs **1 credit per page that returns ≥1 result** (25 companies/page) — so count-first reads `total_count` for at most 1 credit, and a 2,000-company pull costs ~80 credits. The exact same result page requested again within **30 days** is free (`"free": true`). Set `PROSPEO_API_KEY=your_key_here`.

## Output

A deduped, description-confirmed company TAM (domain + name + description) with the final ICP-fit count reported against the raw `total_count` from step 4.
