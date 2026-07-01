# Segment by a custom fit signal

**Goal:** Grade your whole market by any custom fit signal you choose, bucket it into named segments, count each segment for free, and rank best-fit first — before spending a credit pulling the list.

## When to use

Use this when "in my TAM" isn't precise enough and you need to *rank* the market by a fit dimension the Prospeo dashboard won't let you slice cleanly. The lead example: you sell to companies in a size band, but the real buying signal is **how big their engineering org is inside that band** — a 300-person company with 120 engineers is a different buyer than a 300-person company with 12. You pick the signal (engineering density, sales-team size, compliance posture, AI-native, hiring velocity, tech stack, funding stage), define the buckets, and get an ordered, counted, best-fit-first market. This is control the UI cannot give you: a free, exact, multi-segment count sized before any pull.

## Prompt for Claude Code

```
Use the Prospeo MCP server (PROSPEO_API_KEY=your_key_here) to segment my
market by a CUSTOM FIT SIGNAL and rank the segments best-fit first.

My market (edit these): companies with headcount 201-1000, HQ in United
States + United Kingdom + Canada, in Software Development / IT Services.

My custom fit signal (edit this): ENGINEERING DENSITY — the size of the
engineering org inside that company-size band. I want the market split
into fit tiers by engineering headcount, not just total headcount.

Do this:
1. Resolve any non-enumerable values (locations, industries) verbatim
   through /search-suggestions first — 0 credits. Never send raw strings.
2. Anchor the size band once with company_headcount_range (or
   company_headcount_custom) so every segment shares the same company-size
   floor/ceiling.
3. Define 3-4 named fit buckets over the signal. For engineering density,
   layer company_headcount_by_department with department "Engineering &
   Technical" and non-overlapping min/max per bucket, e.g.:
     - Tier A (best): Engineering min 100
     - Tier B: Engineering 40-99
     - Tier C: Engineering 15-39
     - Tier D (thin): Engineering 1-14
4. For EACH bucket, POST /search-company page 1 and read only
   pagination.total_count. Do NOT paginate yet — counting is free
   (a page with 0 results costs 0; you only spend when you pull rows).
5. Print a table: bucket name | filter | total_count, sorted best-fit
   first (Tier A on top).
6. Sanity-check the buckets sum to roughly the un-segmented band count,
   then recommend which tiers to actually pull.

Only after I confirm which tiers to pull, paginate those buckets
(page 1..total_page, max 1000), collect results[].company, and dedupe
by the root `domain` field. Re-confirm ICP fit from each company's
returned description_ai / description_seo, not the company_industry tag.
```

Swap the signal in steps 2-3 for any other fit dimension: sales-team size (`company_headcount_by_department` department `Sales`), hiring velocity (`company_job_posting_quantity` min/max bands), compliance (`company_attributes.has_soc2` / `has_iso27001`), AI-native (`company_type.is_mainly_ai`), funding stage (`company_funding.stage`), or growth (`company_headcount_growth` per-department % bands).

## Prospeo filters / endpoints used

Exact params from the Prospeo TAM Build Guide:

- **`/search-suggestions`** — resolve `location_search` and `industry_search` values verbatim. **0 credits.**
- **`/search-company`** — one call per bucket; read `pagination.total_count` (and `total_page`) from the page-1 response.
- **`company_headcount_range`** (Free) — predefined bands, e.g. `["201-500","501-1000"]`. Mutually exclusive with `company_headcount_custom`.
- **`company_headcount_custom`** (Free) — `{min,max}` for an exact company-size floor/ceiling instead of the preset bands.
- **`company_headcount_by_department`** (Free) — the segmenting lever: array `[{department,min,max}]`, max 10, department `"Engineering & Technical"`. Give each bucket a non-overlapping `min`/`max` (use `null` for open-ended, e.g. `{"department":"Engineering & Technical","min":100,"max":null}`).
- **`company_location_search`** (Free) — `{include,exclude}`, max 100, values via suggestions.
- **`company_industry`** (Free) — `{include,exclude}`; keep it as a wide net, then re-classify from `description_ai` / `description_seo` (industry tags undercount).
- Returned fields used post-pull: `pagination.total_count`, `domain` (dedupe key), `description_ai` / `description_seo` (fit re-check).

Alternate signal levers (same pattern, swap into steps 2-3): `company_headcount_by_department` department `"Sales"`; `company_job_posting_quantity` `{min,max}` (0–5,000, Starter+); `company_attributes` booleans (`has_soc2`, `has_iso27001`, `uses_ai` — Free); `company_type.is_mainly_ai` (Free); `company_funding.stage` (Starter+); `company_headcount_growth` `{timeframe_month,min,max,departments}` (Free).

## Steps

1. **Scope the signal.** Write down the one fit dimension that actually separates good-fit from bad-fit accounts, and the company-size band it lives inside. For engineering density: band = total headcount `201-1000`, signal = engineering headcount inside it.
2. **Resolve values (free).** Call `/search-suggestions` (`location_search`, `industry_search`) to get canonical strings. 0 credits — arbitrary strings like `"uk"` are rejected.
3. **Anchor the band once.** Set `company_headcount_range` (or `company_headcount_custom`) + `company_location_search` + `company_industry`. This shared base is identical across every bucket so the only thing that changes is the signal.
4. **Define 3–4 non-overlapping buckets** over the signal. For engineering density, add `company_headcount_by_department` with department `"Engineering & Technical"` and a distinct `min`/`max` per tier (A: 100+, B: 40–99, C: 15–39, D: 1–14). Non-overlapping ranges keep the counts clean.
5. **Count each bucket free.** POST `/search-company` **page 1 only** per bucket and read `pagination.total_count`. Search bills 1 credit per page that returns ≥1 result — reading total_count off page 1 across a few buckets is at most 1 credit per bucket, and identical requests you already ran in the last 30 days come back `"free": true`.
6. **Rank best-fit first.** Sort the buckets by fit (Tier A on top) into a table: `bucket | filter | total_count`. This ordered, counted market is the deliverable — you now know exactly where the good-fit density sits before pulling anyone.
7. **Sanity-check.** Confirm the buckets roughly sum to the un-segmented band count (run the base filters with no department bucket for the total). Gaps mean overlapping ranges or companies with unknown department headcount.
8. **Pull only the tiers worth it.** For the chosen tiers, paginate `page` 1→`total_page` (cap 1,000 pages / 25,000 rows), collect `results[].company`, dedupe by root `domain`, and re-confirm fit from `description_ai` / `description_seo`.

## Credits

Counting is effectively free: `/search-suggestions` = **0 credits**; `/search-company` = **1 credit per page that returns ≥1 result**, so reading `pagination.total_count` off page 1 costs at most 1 credit per bucket (a page with 0 results costs nothing). Identical searches repeated within **30 days** return `"free": true` at 0 cost. Check budget anytime with `GET /account-information` (free) — `remaining_credits`, `next_quota_renewal_days`. You only spend real credits in step 8, when you paginate the tiers you chose (1 credit / 25 companies).

## Output

An ordered, best-fit-first table of named market segments — `bucket | filter | total_count` — sized for free before any pull, plus deduped company rows (`domain` + `description_ai`) only for the tiers you chose to pull.
