# @pipeworx/ebid

eBid MCP — listings, ended-listing search, bid history and categories from the
eBid.net online marketplace and auction site, via the official eBid Trading API
v2. eBid runs 20+ country sites (us, uk, ca, au, de, fr, …); it is a small
marketplace next to eBay, so treat it as a secondary listings/price source.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1576+ live data sources.

## Tools

- `ebid_search(keyword?, site?, category_id?, min_price?, max_price?, condition?, buy_now_only?, closed?, username?, country?, search_descriptions?, sort?, start?, rows?)` — search live listings, or ended ones with `closed: true` (where realized sale prices live). Max 75 rows/page (API cap).
- `ebid_listing(listing_id, site?, ship_to?)` — one listing in full, with shipping calculated to `ship_to`.
- `ebid_listing_bids(listing_id, site?)` — bid history; on an ended auction the highest bid is the realized price.
- `ebid_categories(category_id?, site?)` — category tree with per-category item counts.

## Auth

- **BYO only:** every eBid API command — including the documented no-login read
  commands — is rejected upstream without an AppId/AppKey pair (error 003).
  Register free at the eBid developer portal <https://ebid.3scale.net/>, then
  pass both as one argument: `_apiKey: "AppId:AppKey"`.
- **Registering is not the same as being able to call.** eBid issues an
  application's AppId and AppKey at signup but activates the application
  separately, so a freshly minted pair authenticates and still answers nothing:
  error 116 `application is not active` on every command. Pipeworx holds such a
  pair (fleet #1284, minted 2026-09-07) and the pack therefore stays BYO-only —
  `PLATFORM_EBID_KEY` is deliberately NOT declared until a live call succeeds,
  because a declared key that cannot answer makes the router keep choosing a
  pack that is guaranteed to fail.

### Telling the four credential failures apart

eBid reports all of them at HTTP 200, and three share code 116. Probed live
2026-09-07 against `api.ebid.net/trading`:

| What you sent | Code | Upstream description |
|---|---|---|
| no AppId | 003 | `AppID not received. Please check your credentials are correctly formatted` |
| an AppId eBid never issued | 116 | `application with id="…" was not found` |
| a real AppId, wrong AppKey | 116 | `application key "…" is invalid` |
| a real, matched, unactivated pair | 116 | `application is not active` |

The pack maps each to its own message. Only the first one tells the caller to
pass a key — the other three mean they already did, and the fix is elsewhere.

## Data sources

- `https://api.ebid.net/trading` — eBid Trading API v2; single endpoint, JSON
  POST with the command in the body. Docs: <https://ebid.3scale.net/doc-json>.
- Failures come back as HTTP 200 with an `Error` array; the pack surfaces the
  upstream code and description rather than the status line.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "ebid": {
      "url": "https://gateway.pipeworx.io/ebid/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/ebid/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1576+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/ebid_search \
  -H 'Content-Type: application/json' \
  -d '{"keyword":"lego","site":"uk"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/ebid_search`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "ebid": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-ebid"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-ebid
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Ebid data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
