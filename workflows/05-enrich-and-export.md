# Enrich, dedupe, export campaign-ready

**Goal:** Turn a qualified account/persona list into a deduped, verified-email, campaign-ready CSV — paying only for verified-email hits.

## When to use

You already have a qualified `/search-person` result set (a list of `person_id`s at your target accounts) and you want contactable, deduped, role-tagged rows ready to drop into a sequencer — without burning credits on no-matches. Example: you scoped a security vendor TAM, pulled security/exec buyers, and now need the outreach-ready CSV.

## Prompt for Claude Code

```
Take my qualified person list (person_ids from the last /search-person pull) and produce a campaign-ready CSV.

1. Bulk-enrich in batches of up to 50 records/request via /bulk-enrich-person.
   Set only_verified_email:true so I pay only on verified-email hits (NO_MATCH is not charged),
   and rely on the 90-day enrich dedup (free_enrichment:true = not charged).
   Do NOT set enrich_mobile unless I say a row needs a phone.
2. Dedupe the enriched rows by root domain (the company `domain` field), not by name or website.
3. Re-classify each company from its returned description_seo / description_ai (and keywords[]),
   NOT from the company_industry tag — drop rows that don't actually fit the ICP.
4. Export a CSV: full_name, job_title, seniority, department, verified_email, company_name,
   domain, company_industry (kept only as metadata), person_location, linkedin_url.
5. Then draft role-based outreach: group the kept rows by seniority/department and write a
   short first-touch per role cluster (e.g. C-Suite vs Director vs practitioner), referencing the
   re-classified company description — no timeline runbooks, no vendor-audit checklists.

Pre-flight GET /account-information first and print remaining_credits so I can confirm the budget.
Keys: PROSPEO_API_KEY=your_key_here (X-KEY header, host https://api.prospeo.io).
```

## Prospeo filters / endpoints used

Everything below runs inside Claude Code via the Prospeo MCP server + Prospeo API key. Note: the MCP exposes only `search` / `enrich` / `account` — the `/bulk-*` endpoints are **REST-only**, so bulk enrich goes through raw REST (`X-KEY: your_key_here`, `Content-Type: application/json`, host `https://api.prospeo.io`).

- **`GET /account-information`** — free, no params. Returns `remaining_credits`, `used_credits`, `next_quota_renewal_days`. Pre-flight budget check.
- **`POST /bulk-enrich-person`** — enriches **up to 50 records per request**, matched on `person_id` (from search). Params:
  - `only_verified_email: true` — spend credits only on verified-email hits; `NO_MATCH` is **not charged**.
  - `enrich_mobile: true` — optional, **10 credits per mobile found** (email included free); omit unless a row needs a phone.
  - Response `free_enrichment: true` = you were **not charged** (90-day enrich dedup on the same record).
- **`company` (API-only) `.websites.include` / `.names.include`** — optional pre-filter (max 500 root domains/names per call) if you re-run `/search-person` to top up the list before enriching.
- **Person fields used for grouping/export** (returned by search/enrich, not filters): `person_id`, `person_seniority`, `person_department`, `person_location_search` value, plus company `domain`, `description_seo`, `description_ai`, `keywords[]`.

## Steps

1. **Pre-flight budget.** `GET /account-information`; read `remaining_credits` and `next_quota_renewal_days`. Confirm you have enough for `~1 credit per verified-email hit` (see Credits).
2. **Batch the IDs.** Split the `person_id` list into groups of **50** (the `/bulk-enrich-person` per-request cap).
3. **Bulk-enrich each batch.** `POST /bulk-enrich-person` with `only_verified_email:true`. No charge on `NO_MATCH`; `free_enrichment:true` rows were free (90-day dedup). Skip `enrich_mobile` unless a row explicitly needs a phone.
4. **Collect verified rows.** Keep only rows that returned a verified email; carry `full_name`, `job_title`, `seniority`, `department`, `verified_email`, company `name`, `domain`, `description_seo`/`description_ai`, `person_location`, `linkedin_url`.
5. **Dedupe by domain.** Dedupe on the root `domain` field (e.g. `intercom.io`), not on name or website. Keep the best row per company/person per your rule (e.g. highest seniority, or one per person).
6. **Re-classify from the description.** For each company, confirm ICP fit from `description_seo` / `description_ai` (+ `keywords[]`), **not** the `company_industry` tag (it undercounts and mis-tags). Drop non-fits.
7. **Export the CSV.** Write the campaign-ready file (columns in step 4). `company_industry` stays only as metadata.
8. **Draft role-based outreach.** Group kept rows by seniority/department, then draft a short first-touch per role cluster in Claude Code, anchored to the re-classified company description. No day-by-day runbooks, no vendor-audit checklists — teach the mechanics, don't prescribe a ritual.

## Credits

- Search that seeded the list = **1 credit per page of ≤25 results** (already spent upstream).
- **Bulk-enrich-person = 1 credit per verified email found.** `only_verified_email:true` means **no charge on `NO_MATCH`** — you pay for hits only.
- **Mobile (optional) = 10 credits per mobile found** (email included free). Omit `enrich_mobile` to avoid it.
- **90-day enrich dedup:** re-enriching the same record inside 90 days is free (`free_enrichment:true`).
- **Worked example:** enriching **1,000** `person_id`s with `only_verified_email:true`, of which **700** return a verified email → **700 credits** (the 300 no-matches cost nothing). Add mobiles on 100 of them → **+1,000 credits**. Re-running the same 1,000 within 90 days → **0 credits**.

## Output

A deduped, ICP-re-classified, verified-email CSV (`full_name, job_title, seniority, department, verified_email, company_name, domain, company_industry, person_location, linkedin_url`) plus per-role first-touch drafts — ready to load into your sequencer.
