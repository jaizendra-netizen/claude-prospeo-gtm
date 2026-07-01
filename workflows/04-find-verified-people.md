# Find verified people in-account

Goal: turn a list of qualified account domains into a persona-targeted people list of verified-email contacts.

## When to use

Use this after you have a deduped list of qualified account domains (the output of a company/TAM build) and you now need the actual buyers inside those accounts. This step feeds the domains back into `/search-person`, layers persona filters (title / seniority / department), keeps only people with a verified email, and caps how many contacts you take per company so the list spreads evenly instead of piling onto a few large accounts. Search returns masked contact details plus a `person_id` you can enrich later — no email credits are spent here.

## Prompt for Claude Code

```
Use the Prospeo MCP server (PROSPEO_API_KEY=your_key_here) to find verified-email buyers
inside a list of qualified account domains.

Input: a deduped list of root domains (e.g. a security vendor TAM, ≤500 domains per batch).

Do this:
1. Resolve every persona title and location to Prospeo's canonical strings with
   /search-suggestions (job_title_search, location_search) — 0 credits. Never send
   arbitrary strings; use the verbatim values it returns.
2. Count-first: POST /search-person page 1 with the filters below and read
   pagination.total_count before pulling anything.
3. Feed the domains back via the API-only company.websites.include filter
   (max 500 domains per call — batch larger lists).
4. Apply persona filters:
   - person_job_title.boolean_search: "(CISO OR \"Head of Security\" OR \"VP Security\" OR CTO) AND !Intern"
   - person_seniority.include: ["C-Suite","Vice President","Head","Director"]
   - person_department.include: ["Information Technology","Engineering & Technical"]
5. Keep verified emails only:
   person_contact_details = {"email":["VERIFIED"],"operator":"OR"}
6. Cap per account: max_person_per_company: 3
7. Paginate page 1 → total_page (max 1,000 = 25,000 results/search), throttling on the
   x-minute-request-left / x-daily-request-left headers; back off on 429.
8. Collect results[].person, dedupe by person_id, and output name, title, company domain,
   seniority, department, and person_id (for a later enrich pass).

Do NOT enrich or spend email credits in this step — search only returns masked details.
```

## Prospeo filters / endpoints used

Endpoints:
- `POST /search-person` — the people pull (company filters are allowed here to constrain which companies the people work at).
- `POST /search-suggestions` — resolve titles/locations to canonical values (0 credits, ≥2 chars, exactly one `*_search` param per call).
- `GET /account-information` — pre-flight credit check (free).

Filters (exact param names):
- `company` (API-only) — `{ "websites": { "include": [ ... ] } }`, root domains only, max 500 per call. This is the lever that scopes the search to your qualified accounts and is not available in the dashboard UI.
- `person_job_title` — `{ "boolean_search": "..." }` (paren depth 5, `!` to exclude, `"exact"` phrases); mutually exclusive with `include`/`exclude`/`match_mode`.
- `person_seniority` — `{ "include": [...] }`, max 11. Enum: `Founder/Owner`, `C-Suite`, `Partner`, `Vice President`, `Head`, `Director`, `Manager`, `Senior`, `Entry`, `Intern`.
- `person_department` — `{ "include": [...] }`. 14 mains (submitting a main auto-includes its subs), e.g. `Information Technology`, `Engineering & Technical`, `Sales`, `Marketing`, `C-Suite`.
- `person_contact_details` — `{ "email": ["VERIFIED"], "operator": "OR" }`. `email` accepts only `["VERIFIED"]`; `operator` is `OR` (default) or `AND`; optional `hide_people_with_details_already_revealed`.
- `max_person_per_company` — int `1–100`; caps people returned per company so contacts spread across accounts.

Response shape: `results[].person` (masked email/mobile + a `person_id`) and `pagination { current_page, per_page: 25, total_page, total_count }`.

## Steps

1. **Start from qualified domains.** Take the deduped root-domain list from your company build. Batch into groups of ≤500 (the `company.websites.include` cap).
2. **Resolve persona values (free).** Call `/search-suggestions` with `job_title_search` for each title and `location_search` for any geo, using exactly one `*_search` param per call. Use the returned canonical strings verbatim.
3. **Count-first (free sizing).** POST `/search-person` page 1 with the full filter set and read `pagination.total_count` to size the persona list before spending search credits.
4. **Scope to the accounts.** Set `company.websites.include` to the batch of qualified domains (root domains only).
5. **Layer persona filters.** Add `person_job_title.boolean_search`, `person_seniority.include`, and `person_department.include` to target the exact buyers.
6. **Keep verified emails only.** Add `person_contact_details = {"email":["VERIFIED"],"operator":"OR"}` so the set contains only contactable people.
7. **Cap per account.** Set `max_person_per_company` (e.g. 3) so no single large account dominates the list.
8. **Paginate.** Loop `page` 1 → `total_page` (max 1,000 pages, so 25,000 results per search). Throttle from `x-minute-request-left` / `x-daily-request-left`; back off on `429` and wait `x-minute-reset-seconds`. Repeat per domain batch.
9. **Dedupe and hand off.** Collect `results[].person`, dedupe by `person_id`, and keep `person_id` for a later `/bulk-enrich-person` pass (up to 50 records/request, 1 credit per verified email found).

## Credits

`/search-person` costs **1 credit per page that returns ≥1 result** (25 results/page, so 1 credit buys up to 25 people). `/search-suggestions` and `/account-information` are **free (0 credits)**. This step only returns masked details — **no email credits are spent** until you run a separate enrich pass (`/bulk-enrich-person` = 1 credit per verified email found, 10 per mobile). Identical search pages re-run within **30 days** are free (`"free": true`). Size the pull first with `pagination.total_count` and pre-flight your balance with `GET /account-information`.

## Output

A deduped, persona-filtered list of verified-email contacts — `name`, `title`, `company domain`, `seniority`, `department`, and `person_id` — scoped to your qualified accounts and ready for a `/bulk-enrich-person` pass.
