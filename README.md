# @pipeworx/bug-bounty-programs

Public bug-bounty PROGRAM DIRECTORY data — program name, bounty range, status
and scope assets — from Bugcrowd, YesWeHack and HackerOne's own public
directories. No vulnerability content, no researcher profiles or handles.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1863+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `bounty_programs_search(platform?, query?, min_bounty?, launched_since?, has_wildcard?, limit?)` — search across one or all three platforms by name, minimum bounty (USD), launch date (HackerOne only — see below), and whether scope includes a wildcard asset like `*.example.com` (checked live on yeswehack/hackerone; bugcrowd returns a `count: 0` with an explanatory `note` instead of an error, because its scope is login-walled, not broken).
- `bounty_program(platform, handle)` — one program by platform + handle. YesWeHack returns a full severity x asset-value reward table and real scope; HackerOne returns real scope but no numeric reward table (not exposed anywhere on the public schema); Bugcrowd returns neither and says why.
- `bounty_new_programs(days)` — programs launched in the last N days, by real launch date. HackerOne only; see "What's not here" below for why Bugcrowd/YesWeHack can't contribute to this one honestly.

## Auth

Keyless. Every endpoint below answers with no account, token, or API key.

## Data sources

- `https://bugcrowd.com/engagements.json` — paged (`?page=N`, 24/page, `paginationMeta.totalCount`) public program list: name, URL, reward summary, industry, access status. **No scope/target data** — Bugcrowd's scope "reveal" action redirects to `/h/engagements/{handle}/reveal.json`, its researcher-sign-in area. That is a genuine login wall, not a bug, and `bounty_program`/`has_wildcard` say so rather than returning a silent empty result.
- `https://api.yeswehack.com/programs` (list, `?page=N`) and `https://api.yeswehack.com/programs/{slug}` (detail) — the richest of the three. Detail returns `scopes[]` (real asset identifiers, some with wildcards), `out_of_scope`, and `reward_grid_default/low/medium/high/critical/very_low` — a real severity x asset-value-tier reward table (only the tiers the program actually uses have non-null values; everything else is `null`, not zero). No launch-date field anywhere on this endpoint — only `last_update_at` (last change, never surfaced as `launched_at` here to avoid implying a launch).
- `https://hackerone.com/graphql` (POST, unauthenticated) — `query { teams }` / `query { team(handle) }`. **Introspection is disabled**, so every field name below was found by probing a guess and reading GraphQL's own "did you mean `X`?" error back, not from a published schema doc:
  - `teams(first/last, after/before)`: `handle name offers_bounties submission_state launched_at` — `launched_at` is a REAL, verified launch date (confirmed against `hackerone.com/security` → `2013-11-06`, the platform's own founding program). `last: N` returns the N most-recently-created teams, which is what `bounty_new_programs` walks backward from; there is no working `order_by` value we could find (`order_by: {field: ..., direction: DESC}` is accepted syntactically but every field name we tried for `field` was rejected), so sorting relies on sequential internal IDs rather than an explicit date sort.
  - `team(handle).structured_scopes_search(first)`: a union (`DocumentUnion`) — use `... on StructuredScopeDocument { identifier asset_type eligible_for_bounty eligible_for_submission instruction }`. `identifier` is the actual scope asset (domain/URL), sometimes a wildcard (`*.rubyonrails.org`, `*.cloudflarepartners.com` confirmed live). **No numeric bounty amount anywhere we could find** — `team.bounty_table` exists as a type but every plausible field name on it (`rewards`, `severities`, `low_bounty`/`high_bounty`, `min`/`max`, `field_names`, …) came back `doesn't exist`. If HackerOne ever documents the real field names, `bountyProgram()`'s `hackerone` branch is the one place to add them.

## What's not here: Immunefi

Immunefi is **not** a platform in this pack, on purpose. `/explore` and every
`/bug-bounty/{slug}/` page are public — no login needed — but the actual scope
list and severity-tiered reward table load client-side from an internal API
that is not reachable as a stable, documented JSON endpoint. Checked and ruled
out, 2026-10-08:

- Server-rendered HTML: the full page (200, ~210KB for `/bug-bounty/ens/`) carries
  zero reward-table fields and exactly one `0x...` contract address — the scope
  list renders after hydration, client-side.
- The Next.js RSC flight payload (`RSC: 1` header, the app-router equivalent of
  Next's old `_next/data/*.json`): same result, no reward/scope keys in the
  ~30KB response.
- `robots.txt` / `sitemap-dynamic.xml`: public and unauthenticated, and useful
  for *discovering* program slugs (`/bug-bounty/{slug}/` × ~166), but a sitemap
  only carries `lastmod` (last content change), never a launch date.
- The one genuinely public, static fact per program is the "rewards up to $X"
  line Immunefi bakes into the page's SEO meta description tag — real,
  but a single top-line number is too thin to ship as a program record next to
  the other three platforms' structured scope + reward tables.

If Immunefi ever publishes (or we find) a stable JSON endpoint for this, it's a
fourth `platform` value to add, following the same shape as `yeswehack`.

## Also not here: Intigriti

Not checked — Intigriti's public directory requires a researcher account/token
to browse, per the task that specified this pack. Skipped rather than scraped
from behind a login.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "bug-bounty-programs": {
      "url": "https://gateway.pipeworx.io/bug-bounty-programs/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/bug-bounty-programs/mcp` returns the tools in the table
above **plus the shared Pipeworx meta-tools** — `ask_pipeworx`,
`discover_tools`, `search_within`, `remember`/`recall` and the rest of the
gateway-wide set. So the tool count you see is larger than this table: a
single-pack endpoint currently lists roughly 30 shared tools alongside the
pack's own. The connection's `initialize` response states its exact scope, and
is the authoritative answer for a given day.

This is deliberate, not multiplexing by accident. The meta-tools are what let a
scoped connection answer a question this pack does not cover — via
`ask_pipeworx`, which routes across the whole catalog — without you adding a
second MCP server. There is currently no way to mount a pack endpoint without
them; if the extra schemas cost you more context than the routing is worth,
connect to the full gateway once rather than to several pack endpoints.

Or connect to the full Pipeworx gateway to get every pack's tools listed
directly, instead of just this one's:

```json
{
  "mcpServers": {
    "pipeworx": {
      "url": "https://gateway.pipeworx.io/mcp"
    }
  }
}
```

Both URLs reach the same gateway and the same 1863+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/bounty_programs_search \
  -H 'Content-Type: application/json' \
  -d '{"platform":"yeswehack","min_bounty":1000,"limit":10}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/bounty_programs_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-bug-bounty-programs` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/bug-bounty-programs/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "bug-bounty-programs": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-bug-bounty-programs"]
    }
  }
}
```

Without a key it calls anonymously, capped at 50 calls a day. A free account
raises that to 200 a day: get a key at https://pipeworx.io/signup and add
`"env": { "PIPEWORX_API_KEY": "<your key>" }` to the server entry.
The key is sent as a Bearer token, the same way a hosted connection sends it,
so calls are metered and billed exactly as they are there.

It lists **only** this pack's tools by default. Set `PIPEWORX_META_TOOLS=1` to
also list `ask_pipeworx` and the other gateway-wide tools.

Behind an HTTPS proxy: Node's built-in `fetch` does not read `HTTPS_PROXY` by
default. Newer Node releases honour it when `NODE_USE_ENV_PROXY=1` is set; check
your Node version's documentation.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Bug Bounty Programs data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
