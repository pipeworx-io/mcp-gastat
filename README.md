# @pipeworx/gastat

Saudi Arabia's official statistics — GASTAT (General Authority for Statistics)
population, employment/unemployment, GDP, consumer prices and trade
indicators, each carrying both Arabic and English titles under one indicator
id.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1683+ live data sources.

## Tools

- `gastat_headline()` — quick snapshot of GASTAT's own homepage indicators
  (population, GDP, unemployment rate, growth rate, per-capita income).
- `gastat_search(term, limit?)` — find indicators by keyword in English or
  Arabic across ~2,600 indicators; returns indicator ids + bilingual titles.
- `gastat_series(indicator_id, recent?)` — full time series for an indicator
  id (from `gastat_search` / `gastat_headline`) — multiple periods for
  comparison over time, not just a latest-value lookup.

## Auth

Keyless. Every endpoint here is served to anonymous visitors with no login,
token, or registration — verified live.

## Data sources

- `https://database.stats.gov.sa/gastatapi/portal/api/v1/indicators/cardsinfo`
  — headline indicators (population, GDP, unemployment, etc.).
- `https://database.stats.gov.sa/gastatapi/portal/api/v1/indicators/public`
  — the full indicator catalog (~2,600 REP nodes), ~4.5MB, no server-side
  filtering. This is the same endpoint GASTAT's own Angular app
  (`database.stats.gov.sa`) fetches and searches client-side; there is no
  documented developer portal, so this was found by reading the app's own JS
  bundle rather than any published API docs. `gastat_search` and
  `gastat_series` both fetch it fresh per call (no cross-request cache
  available without a binding) — expect a few seconds of latency, that's
  GASTAT's shape, not ours.
- `https://database.stats.gov.sa/gastatapi/portal/api/v1/indicators/getDataForChart?api=<token>`
  — the actual observation data for one indicator, where `<token>` is the
  opaque `api_url` field carried on that indicator's catalog entry (stable
  across repeated fetches in testing — not session-bound).

**Trap:** most catalog entries carry a bare title + `api_url` token that
resolves cleanly with no extra parameters. A meaningful minority — anything
broken out by region, nationality, sector, or commodity in its title — needs
dimension filters this endpoint doesn't expose through a plain GET, and comes
back either an HTTP 400 or a row of all-null values. `gastat_series` detects
both cases and returns a clear `error`/`message` rather than surfacing the
raw failure, and `gastat_search` ranks indicators without a breakdown hint in
their title first, since those are the ones that resolve.

**Bilingual trap:** GASTAT indexes everything by Arabic AND English title
under one indicator id — never two ids for the same series. Every tool here
preserves both `title_en` and `title_ar` on the same record rather than
treating the Arabic filing as a separate entity.

Query-string parameters on `indicators/public` (e.g. `?term=`) are rejected by
GASTAT's own WAF — confirmed live. There is no server-side search; the whole
catalog must be fetched and filtered client-side, which is what
`gastat_search` does.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "gastat": {
      "url": "https://gateway.pipeworx.io/gastat/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/gastat/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1683+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/gastat_headline \
  -H 'Content-Type: application/json' \
  -d '{}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/gastat_headline`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "gastat": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-gastat"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-gastat
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Gastat data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
