# @pipeworx/worldpop

WorldPop (University of Southampton) 100 m gridded population estimates,
2000-2020, summed over any polygon you supply. Answers "how many people live
inside this boundary" for arbitrary geometry — a catchment, a flood extent, a
service area — where census tables only answer for official administrative
units.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1684+ live data sources.

## Tools

- `worldpop_stats_polygon(geojson, year?, dataset?, runasync?)` — total
  population inside a GeoJSON Polygon/MultiPolygon.
- `worldpop_datasets()` — the population surfaces the stats service exposes.
- `worldpop_task_status(taskid)` — state and result of one async job.

## Auth

Keyless. WorldPop asks for fair use (roughly 1,000 calls/day) and no
registration.

## Data sources

- <https://api.worldpop.org/v1/services> — dataset list (`wpgppop` population
  count, `wpgpas` age and sex structures).
- <https://api.worldpop.org/v1/services/stats> — submit a polygon job.
- <https://api.worldpop.org/v1/tasks/{taskid}> — collect the result.

Two traps this pack absorbs:

**The stats call is ASYNCHRONOUS and answers `200` either way.** Submitting
returns `{"status":"created","taskid":...}` with no population in it — a caller
that reads that as the answer gets a silent zero. `worldpop_stats_polygon`
polls `/tasks/{id}` until `status` is `finished` (12 attempts, ~1.5 s apart;
a city-sized polygon finishes in about 3 s) and returns the taskid regardless,
so nothing is lost if the polling window expires.

**A GeoJSON `Feature` is rejected.** WorldPop's validator accepts only a bare
geometry and answers `"Invalid GeoJSON: ... is not valid under any of the given
schemas"` — as a `200` on the task, with `error: true`, not as an HTTP error.
Every mapping tool hands you a Feature, so `normaliseGeometry` unwraps
Feature and FeatureCollection before the call.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "worldpop": {
      "url": "https://gateway.pipeworx.io/worldpop/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/worldpop/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1684+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/worldpop_stats_polygon \
  -H 'Content-Type: application/json' \
  -d '{"geojson":{"type":"Polygon","coordinates":[[[-0.14,51.5],[-0.1,51.5],[-0.1,51.53],[-0.14,51.53],[-0.14,51.5]]]},"year":2020}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/worldpop_stats_polygon`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "worldpop": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-worldpop"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-worldpop
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Worldpop data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
