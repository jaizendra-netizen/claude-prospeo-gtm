# End-to-end pipeline (one run)

**Goal:** Go from a plain-language ICP to a verified, campaign-ready pipeline in a single Claude Code session — sized TAM, segmented, signal-layered, verified people, enriched, exported — using only Claude Code and Prospeo.

## When to use

Use this when you want the whole system in one shot instead of running each stage by hand. It chains the five workflow templates in this repo into one continuous run:

1. `01-tam-builder.md` — plain-language ICP → sized company TAM
2. `02-segment-market.md` — grade/segment the TAM by fit signals
3. `03-layer-signals.md` — stack funding / intent / news / hiring / growth / lookalike / Google-discovery
4. `04-people-finder.md` — verified personas inside target accounts
5. `05-enrich-and-export.md` — bulk-enrich, dedupe, export + draft outreach

Reach for this once you already know your ICP well enough to describe it in a sentence and just want the pipeline built. If you're still exploring the market, run `01` and `02` on their own first, then come back here.

## Prompt for Claude Code

```
You are running the end-to-end Prospeo pipeline (template 06) inside Claude Code.
Drive Prospeo through the Prospeo MCP server for single searches / enrich / credit
checks, and raw REST (host https://api.prospeo.io, header `X-KEY: your_key_here`)
for /search-suggestions and the /bulk-enrich-* endpoints the MCP does not expose.

MY ICP (plain language):
<one or two sentences — e.g. "AI-native security vendors in the US, UK and Canada,
Series A–C, 50–1,000 employees, with a SOC 2 signal, and I want their CISO / Head of
Security / CTO with verified emails.">

Run these stages in order and pause after each with a one-line summary + the credit
spend so far:

1. TAM BUILD (see 01-tam-builder.md)
   - Resolve every non-enumerable value (locations, industries, titles, technologies,
     investors, ICP fields) verbatim via POST /search-suggestions (0 credits).
   - Count-first: POST /search-company page 1 with a MULTI-SIGNAL net
     (company_location_search + company_industry + company_keywords), read
     pagination.total_count. NEVER seed on company_industry alone.
   - Report total_count before pulling.

2. SEGMENT (see 02-segment-market.md)
   - Tighten with company_headcount_range / company_headcount_custom,
     company_headcount_by_department, company_revenue, company_type
     (status / subtypes / business_model / is_mainly_ai), company_attributes
     (uses_ai, has_soc2, has_iso27001…). Count each segment for free before pulling.

3. LAYER SIGNALS (see 03-layer-signals.md)
   - Stack the levers that fit the ICP: company_funding (stage / funding_date),
     company_intent, company_news, company_headcount_growth,
     company_job_posting_hiring_for, company_lookalike (seed domain),
     company_google_discovery / company_website_search (Pro).
   - If total_count > 25,000, split into non-overlapping sub-searches and union.

4. PEOPLE (see 04-people-finder.md)
   - Paginate the qualified companies, dedupe by the returned `domain`, re-classify
     each from description_seo / description_ai (NOT the industry tag).
   - Feed qualified domains back via company.websites.include (≤500/call) to
     /search-person with person_job_title (boolean_search), person_seniority,
     person_department, person_contact_details.email:["VERIFIED"], and
     max_person_per_company to spread evenly.

5. ENRICH + EXPORT (see 05-enrich-and-export.md)
   - GET /account-information first to budget. Bulk-enrich via /bulk-enrich-person
     (≤50 records/request) on person_id, only_verified_email:true, enrich_mobile
     only where essential. Export a campaign-ready CSV, then draft role-based
     outreach.

Rules: at least one positive (include) filter per search; use /account-information
to pre-flight credits; respect the live rate-limit headers and back off on 429.
```

## Prospeo filters / endpoints used

Everything below is grounded in `PROSPEO_TAM_BUILD_GUIDE.md`.

**Endpoints:** `POST /search-suggestions` (0 credits), `POST /search-company`, `POST /search-person`, `POST /bulk-enrich-person`, `GET /account-information` (free). Host `https://api.prospeo.io`, auth header `X-KEY`, `Content-Type: application/json`, body `{ "filters": { … }, "page": N }`.

**MCP vs REST:** the `prospeo` / `prospeo-hosted` MCP servers expose only `search` + `enrich` + `account`. `/search-suggestions` and the `/bulk-*` endpoints are **not** on the MCP — call those via raw REST.

**Stage 1 — TAM (`/search-company`):** `company_location_search`, `company_industry`, `company_keywords` (`sources`: specialties, seo_description, ai_description, website_pages). Read `pagination.total_count`.

**Stage 2 — Segment:** `company_headcount_range`, `company_headcount_custom`, `company_headcount_by_department`, `company_revenue` (Starter+), `company_type` (`status`, `subtypes`, `business_model`, `is_mainly_ai`), `company_attributes` (`uses_ai`, `has_soc2`, `has_iso27001`).

**Stage 3 — Signals:** `company_funding` (`stage`, `funding_date`, `last_funding`, `total_funding`, `investors`), `company_intent` (`topic_ids`), `company_news` (`categories`, `timeframe_days`), `company_headcount_growth`, `company_job_posting_hiring_for` (`boolean_search`), `company_lookalike` (`domain` seed, `minimum_tier`), `company_google_discovery` (Pro+, `seo_keywords`), `company_website_search` (Pro+).

**Stage 4 — People (`/search-person`):** `company.websites.include` (API-only, ≤500/call), `person_job_title` (`boolean_search`), `person_seniority`, `person_department`, `person_contact_details` (`email`: `["VERIFIED"]`, `operator`), `max_person_per_company`. Returned fields for re-classification: `domain`, `description_seo`, `description_ai`.

**Stage 5 — Enrich (`/bulk-enrich-person`):** ≤50 records/request keyed on `person_id`, `only_verified_email: true`, `enrich_mobile: true` (optional). Pre-flight with `GET /account-information` (`remaining_credits`, `next_quota_renewal_days`).

## Steps

1. **Scope from the ICP sentence.** Extract industry / sub-niche, size, revenue, geo, funding, public/private, persona (titles / seniority / department), and which signals to layer.
2. **Resolve values (free).** `POST /search-suggestions` (0 credits, one `*_search` param per call, ≥2 chars) to translate every human phrase into Prospeo's canonical strings/codes.
3. **Count-first TAM.** `POST /search-company` page 1 with a multi-signal net; read `pagination.total_count`. Never seed on `company_industry` alone — it undercounts.
4. **Segment for free.** Add size / revenue / type / attribute filters and re-read `total_count` per segment before spending any credits.
5. **Layer signals.** Stack funding / intent / news / growth / hiring / lookalike / Google-discovery to match the ICP. If `total_count > 25,000` (max 1,000 pages × 25), split into non-overlapping sub-searches and union.
6. **Paginate + dedupe.** Loop `page` 1 → `total_page` (cap 1,000), collect `results[].company`, dedupe by the root `domain`, and re-classify each company from `description_seo` / `description_ai`, not the industry tag.
7. **Find verified people.** Batch qualified domains (≤500) into `/search-person` with the persona filters and `person_contact_details.email:["VERIFIED"]`; cap with `max_person_per_company`.
8. **Enrich in bulk.** `GET /account-information` to budget, then `/bulk-enrich-person` (≤50/request) on `person_id` with `only_verified_email: true`; add mobiles only where essential.
9. **Export + hand off.** Write the campaign-ready CSV and draft role-based outreach — all inside Claude Code.

## Credits

- `/search-suggestions` and `/account-information` are **free (0 credits)** — resolve values and size TAMs before spending.
- Search = **1 credit per page that returns ≥1 result** (25 results/page ⇒ 1 credit buys up to 25 companies or people).
- Enrich-person = **1 credit per email found**; `enrich_mobile` = **10 credits per mobile** (email included). No charge on `NO_MATCH`.
- **Free repeats:** identical search pages re-run within **30 days** are free (`"free": true`); re-enriching the same record within **90 days** is free (`free_enrichment: true`).
- Example: a 2,000-company TAM ≈ 80 search credits; a 5,000-person persona pull ≈ 200 search credits; enriching 5,000 verified emails ≈ 5,000 credits.

## Output

A deduped, ICP-verified account list plus per-account verified-email contacts (optional mobiles) exported as a campaign-ready CSV with role-based outreach drafted — produced end to end inside Claude Code from a single plain-language ICP.
