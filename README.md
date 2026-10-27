# @pipeworx/meetup

Meetup event and group search MCP — public events, groups and topic categories from Meetup, searchable by keyword, city, or group. Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1882+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `meetup_search_events(query, city?, lat?, lon?, radius?, limit?)` — search upcoming public events by keyword near a city (geocoded automatically) or a lat/lon point. Returns each event's title, time, URL, status, type and hosting group.
- `meetup_event(event_id)` — full details for one event by its numeric Meetup ID (the trailing number in a meetup.com event URL). Returns title, description, start/end time, status, type, venue and hosting group.
- `meetup_group(urlname)` — details for one group by its urlname (the slug right after `meetup.com/`, e.g. "sfpython"). Returns name, description, location, timezone, topic category and founding date.
- `meetup_group_events(urlname, limit?, include_past?)` — a group's events by urlname; upcoming by default, past events with `include_past: true`.

None of these tools request or return attendee/RSVP lists, member lists, or any other personal contact data — deliberate, not an oversight (see "Privacy" below).

## Auth

Keyless. No key, no account, no rate-limit documentation published.

## Data sources

- Meetup (meetup.com), the public GraphQL API behind its own website search and event/group pages — event, group and topic-category metadata. No download file, feed or query shape is named beyond that; see `src/index.ts` for the route.

## Privacy

Meetup's schema exposes attendee RSVPs, host identities and member lists on `Event`/`Group`. None of those fields are selected anywhere in this pack — every query here is scoped to public event/group metadata (title, time, venue, description, location) and nothing else. Keep it that way: don't add `rsvps`, `hostRsvps`, `memberships`, `organizer`, `eventHosts` or `savedBy` to any query without re-reading this note.

## Gotchas

- **`lat`/`lon` are required by the upstream API itself**, not a choice this pack made — `EventSearchFilter`/`GroupSearchFilter` both declare them non-null. `meetup_search_events` resolves a `city` string to coordinates via the shared `resolvePlace` geocoder (`@pipeworx/shared`, Open-Meteo-backed, keyless) before calling Meetup, so a caller never has to supply raw coordinates — but if geocoding fails to resolve a city, the tool fails loudly rather than guessing.
- **`radius` is in miles.** The upstream filter field is a bare `Float` with no documented unit; verified empirically against known US city radii.
- **Event and group IDs are opaque strings, not always numeric-looking everywhere** — `event_id` is the trailing path segment of a `meetup.com/{group-urlname}/events/{event-id}/` URL, and a group's `urlname` is the slug right after `meetup.com/`. Neither is discoverable from the other without a lookup; `meetup_search_events` and `meetup_group_events` are what hand you both.
- **`meetup_group_events` with no events scheduled returns `total_count: 0`, not an error** — groups that are real and active but between events (verified live against `sfpython`) look identical to a dead/wrong urlname from the shape of the response alone; only `meetup_group` erroring on the urlname itself distinguishes "no group" from "no events right now".
- **GraphQL errors come back as HTTP 200** with a top-level `errors` array (standard GraphQL behavior) — this pack checks `errors` explicitly rather than trusting `res.ok`, since a malformed query or a bad ID both return 200.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "meetup": {
      "url": "https://gateway.pipeworx.io/meetup/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/meetup/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1882+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/meetup_search_events \
  -H 'Content-Type: application/json' \
  -d '{"query":"music","lat":40.71,"lon":-74,"radius":20,"limit":5}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/meetup_search_events`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "meetup": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-meetup"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-meetup
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about Meetup data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
