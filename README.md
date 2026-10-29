# @pipeworx/sec-nmfp

US money market fund portfolio holdings, from SEC Form N-MFP. Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1857+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `nmfp_security_holders(issuer, period?, limit?)` — which money market funds hold a given issuer's paper. Accepts an issuer name fragment or a CUSIP.
- `nmfp_fund_holdings(fund, period?, limit?)` — one fund's reported portfolio for a month, with a per-category position count.
- `nmfp_periods()` — which monthly reporting periods SEC has published (currently 50) and the date range each covers.

## The inversion is the point of this pack

SEC publishes N-MFP **filing-first**: one filing per fund per month, portfolio inside. So "what does
this fund hold" is the natural question and *"which funds hold this issuer's paper"* is not — that one
requires reading every filing in the period. `nmfp_security_holders` does exactly that on each call,
which is the reason to have the pack at all. When a bank or an issuer wobbles, the question people
ask is who is exposed, and there is no filing that answers it.

Measured on the 2026-07-09→2026-08-07 period: 341 fund filings, 47,129 holdings. `Federal Home Loan
Bank` alone appears in **129 funds across 6,489 positions**.

## Reading traps, surfaced in the responses

- **Positions are counts, not dollars.** N-MFP position value is not carried here, so
  `positions_by_category` says how many holdings sit in each category, not how much money does. A
  fund running many small repo positions against one large Treasury block reads repo-heavy by count
  and may not be by value. Both tools say this in `reading_note`.
- **Only money market funds file N-MFP.** A bond or equity fund will never appear, however large.
  The `found: false` path says that rather than implying the fund holds nothing.
- **Issuer names are as the fund typed them** — `FEDERAL HOME LOAN BANK SYSTEM`, not a normalized
  entity. Matching is substring, and the miss path tells the caller to shorten the fragment rather
  than concluding no fund holds it.

## Shape

One ZIP per monthly period, ~11.9MB, carrying 23 TSVs. Three are read: `NMFP_SUBMISSION` (the fund
and its filing), `NMFP_SERIESLEVELINFO`, `NMFP_SCHPORTFOLIOSECURITIES` (the holdings). The rest are
skipped — including `NMFP_COLLATERALISSUERS`, which is 55MB raw on its own, five times the whole
rest of the archive, and answers no question asked here.

Read live per call rather than kept in a mirror: one period is one fetch, and a money-market question
is nearly always about the latest month.

Two file-format notes worth keeping, because both have cost time in this codebase:

- Sizes come from the ZIP **central directory**, not local headers. SEC sets the data-descriptor flag
  on some datasets, which zeroes the local-header sizes; a reader that trusts them extracts nothing.
- The filenames encode a **date range** (`20260709-20260807_nmfp.zip`) that is not derivable — the
  windows do not align to calendar months and shift each cycle. This is the fourth SEC dataset here
  with its own path convention, so the index page is read and the URL is never constructed.

## Cadence

Money market funds file monthly; SEC posts a period a few days after it closes.

## Source

SEC Form N-MFP data sets — https://www.sec.gov/data-research/sec-markets-data/dera-form-n-mfp-data-sets

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "sec-nmfp": {
      "url": "https://gateway.pipeworx.io/sec-nmfp/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/sec-nmfp/mcp` returns the tools in the table
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
curl -X POST https://gateway.pipeworx.io/v1/tools/nmfp_security_holders \
  -H 'Content-Type: application/json' \
  -d '{"issuer":"Federal Home Loan Bank","limit":10}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/nmfp_security_holders`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Run it from npm

`@pipeworx/mcp-sec-nmfp` is a local stdio bridge to the Pipeworx gateway: your MCP client
starts it with `npx`, and it forwards every call to the same
`https://gateway.pipeworx.io/sec-nmfp/mcp` endpoint shown above. It needs network access;
there is no offline mode.

```json
{
  "mcpServers": {
    "sec-nmfp": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-sec-nmfp"]
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
ask_pipeworx({ question: "your question about Sec Nmfp data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
