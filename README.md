# @pipeworx/worldpop

WorldPop (University of Southampton) 100 m gridded population estimates,
2000-2020, summed over any polygon you supply. Answers "how many people live
inside this boundary" for arbitrary geometry — a catchment, a flood extent, a
service area — where census tables only answer for official administrative
units.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1857+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

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

Both URLs reach the same gateway and the same 1857+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/worldpop_stats_polygon \
  -H 'Content-Type: application/json' \
  -d '{"geojson":{"type":"Polygon","coordinates":[[[-0.14,51.5],[-0.1,51.5],[-0.1,51.53],[-0.14,51.53],[-0.14,51.5]]]},"year":2020}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/worldpop_stats_polygon`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-worldpop` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/worldpop/mcp` endpoint shown above. It needs network access;
there is no offline mode.

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
ask_pipeworx({ question: "your question about Worldpop data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
