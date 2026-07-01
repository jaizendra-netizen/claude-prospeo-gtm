# Claude Code + Prospeo — project config

This project drives the [Prospeo](https://prospeo.io) API from Claude Code to go from an ICP to a verified, campaign-ready pipeline.

## 1. Set your key

Copy `.env.example` to `.env` and set your Prospeo API key:

```
PROSPEO_API_KEY=your_key_here
```

Prospeo auth is the header `X-KEY: <api_key>` (NOT `Authorization: Bearer`). Host: `https://api.prospeo.io`.

## 2. Wire up the Prospeo MCP server

Hosted (no local process, recommended):

```
claude mcp add --transport http prospeo https://mcp.prospeo.io --header "X-KEY: $PROSPEO_API_KEY"
```

Or local (npx):

```
claude mcp add prospeo --env PROSPEO_API_KEY=$PROSPEO_API_KEY -- npx -y @prospeo/prospeo-mcp-server
```

The MCP exposes `search` + `enrich` + `account`. It does **not** expose `/search-suggestions` or the `/bulk-*` endpoints — for those stages, call the REST API directly (`curl` to `https://api.prospeo.io`, header `X-KEY`).

## 3. Run a workflow

Open Claude Code in this repo and run any file in `workflows/` (start with `workflows/01-icp-to-sized-tam.md`), or run `workflows/06-end-to-end-pipeline.md` for the full ICP → handoff pipeline. Reusable prompt snippets are in `PROMPTS.md`; the complete filter reference is in `docs/FILTER-REFERENCE.md`.

## Guardrails

- **Size before you spend.** `/search-suggestions`, `pagination.total_count` (page 1), and `GET /account-information` are free (0 credits). Always count-first.
- **Search = 1 credit / 25 results.** Enrich pays 1 credit per verified email (mobiles 10 each, opt-in). No charge on `NO_MATCH`.
- Never commit `.env` or any real key.
