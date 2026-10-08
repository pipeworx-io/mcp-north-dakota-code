# @pipeworx/north-dakota-code

North Dakota Century Code — state statutes by citation and by official
catchline (topic search). Keyless.

Part of [Pipeworx](https://pipeworx.io) — an MCP gateway connecting AI agents to 1721+ live data sources. This is an independent, unofficial integration — not affiliated with, endorsed by, or published by the upstream provider.

## Tools

- `nd_statute(citation)` — full text of a North Dakota Century Code section
  by citation ("12.1-16-01" is murder; "47-16-13.1" is a landlord's duty to
  maintain a rental unit). Resolves the citation to its one chapter PDF,
  fetches it live, and extracts the text.
- `nd_search(query, limit?)` — topic/keyword search over the Legislature's
  own official section and chapter catchlines (not full statutory prose —
  see below), returning matching citations.

## Auth

Keyless.

## The survey trap did not fully apply here

The recorded trap for this pack family was "published only as PDF" — true
for the statutory TEXT (ndlegis.gov serves each chapter as a small PDF,
`cencode/t12-1c16.pdf`), but the Legislature also publishes one combined
HTML page listing every title, chapter, and section with its official
catchline:

```
https://ndlegis.gov/general-information/north-dakota-century-code/index.html
```

That page is ~9.5 MB and covers the WHOLE Code (~29,000 sections, 1,740
chapters, 74 titles) in a single document — too large to fetch per call, and
there is no lighter per-title listing page (unlike Iowa's) and no search API
for the Code (ndlegis.gov's bill-tracking side has a JSON API; the Code side
does not, confirmed live). So it is fetched and baked ONCE into
`src/nd-index-data.ts` — citation + official catchline + chapter/title
context, NOT statutory text — by `scripts/bake-index.mjs`, the same "baked
table" shape `mcps/symmap` uses for its herb table. `nd_search` reads that
baked table entirely in memory; `nd_statute` uses it only to resolve a
citation to its one chapter PDF before fetching and extracting that PDF
live, so the statutory text itself is never baked or redistributed.

### Refresh path

Re-run the bake script whenever North Dakota recodifies (new session laws
land continuously; the index page updates on its own schedule):

```
node mcps/north-dakota-code/scripts/bake-index.mjs > mcps/north-dakota-code/src/nd-index-data.ts
```

It fails loudly (not silently) if the page shrinks below ~1 MB or parses to
suspiciously few rows — both would mean ndlegis.gov changed its markup
rather than that the Code got smaller.

## pdf-text.ts needed two real extensions for this state

`shared/src/pdf-text.ts` (built for Kentucky's per-statute PDFs) didn't
handle North Dakota's chapter PDFs out of the box — fixed in a separate
commit, with a test:

- Every content stream writes `/Length` as an INDIRECT reference
  (`/Length 12 0 R`) rather than inlining the literal integer. The
  original detection regex was a backtracking trap for a multi-digit
  referenced object number (it would silently resolve to the literal
  integer 1 instead of following the reference) — rewritten as an explicit
  optional group with nothing to backtrack.
- The body text font is a simple (non-Type0) TrueType subset with **no**
  `/Encoding` at all — unreadable as WinAnsi bytes. Its own `/ToUnicode`
  CMap (keyed by a 1-byte code, unlike Type0's 2-byte CID) is the only
  place the real character is recorded, so the extractor now checks for
  that whenever a simple font isn't declared WinAnsi.
- `/Font` itself is an indirect reference inside `Resources` (`/Font 18 0
  R`), which Kentucky's PDFs never used — resolved the same way `/Pages`
  already was.

## Sectioning without reliable whitespace

Once extracted, a chapter's text has no reliable line break between
sections — headings run directly into the body ("12.1-16-01.
Murder.1.A person is guilty..."). `sliceNdSection` builds its heading
boundaries from the chapter's OWN verified citation list (already known
from the baked index, not a generic number pattern), which rules out
matching a cross-reference to a citation in a different chapter; the usual
most-text-wins tie-break still guards the rare same-chapter cross-reference
that happens to end a sentence at the same number.

## Four capabilities — two available, two are not

- **Citation lookup** — available (`nd_statute`).
- **Topic/keyword search** — available (`nd_search`), but it matches
  official CATCHLINES, not full statutory prose. North Dakota does not
  publish a full-text search of Code section text (verified live). A query
  like "landlord" still surfaces 7 real sections (e.g. "Landlord
  obligations - Maintenance of premises") even though no CHAPTER is named
  "Landlord and Tenant" — the sections are scattered across Titles 28, 31,
  35, and 47, which only the section-level catchline index catches.
- **Amendments/enactment history** — **not available** for an in-force
  section. ndlegis.gov's current online Code shows no "History:"/"Amended
  by" line for a section still in force (verified across a full chapter,
  47-16, Leasing of Real Property: zero such lines across ~40 live
  sections). A REPEALED or REDESIGNATED section's one-line disposition
  ("Repealed by S.L. 1977, ch. 429, § 7.") is the only session-law citation
  the Code pages carry at all, and `nd_statute` returns it as-is in `text`
  rather than suppressing it. Reconstructing real amendment history would
  mean parsing North Dakota's separate Session Laws archive
  (`ndlegis.gov/research-and-archives/session-laws`, one PDF volume per
  legislative session, not indexed by section) — out of scope here.
- **Historical version** (a section's text as it read before an amendment)
  — **not available**. ndlegis.gov's own historical archive
  (`research-and-archives/historical-constitution-and-century-code`) only
  reaches pre-1930 Revised Codes/Compiled Laws (1877-1925) — long before
  Title 12.1 (the Criminal Code, enacted 1973) or most of the modern Code
  existed — and there is no per-year archive of the current-era Code the
  way Iowa or Indiana publish. The current online edition is the only one.

## Citations

North Dakota cites by title-chapter-section, e.g. "12.1-16-01" (Title 12.1,
Chapter 16, Section 01: Murder) or "47-16-13.1" (Title 47, Chapter 16,
Section 13.1: landlord maintenance duty). The chapter and section can each
carry a decimal suffix ("12.1-27.2-02", "47-16-07.3") — `nd_statute` looks
the citation up against the baked index directly rather than parsing it into
parts, so any suffix shape the index itself uses round-trips correctly.

## Data sources

- `https://ndlegis.gov/general-information/north-dakota-century-code/index.html`
  — the title/chapter/section/catchline index for the whole Code. Baked
  once into `src/nd-index-data.ts`; never fetched per call.
- `https://ndlegis.gov/cencode/t12-1c16.pdf` — one chapter's full statutory
  text (here, Title 12.1 Chapter 16 — swap the filename for any other
  chapter). `nd_statute` fetches this live on every call and extracts the
  target section out of it.

Both are plain, keyless, server-rendered HTML/PDF — no login, no JS, no
rate-limit observed during development.

## Quick Start

Add to your MCP client (Claude Desktop, Cursor, Windsurf, etc.):

```json
{
  "mcpServers": {
    "north-dakota-code": {
      "url": "https://gateway.pipeworx.io/north-dakota-code/mcp"
    }
  }
}
```

### What this endpoint actually serves

`tools/list` at `https://gateway.pipeworx.io/north-dakota-code/mcp` returns the tools in the table
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

Both URLs reach the same gateway and the same 1721+ data sources. The
only difference is which pack's tools are listed **directly**; `ask_pipeworx`
reaches all of them from either one.

## No MCP client? Call it over HTTP

```bash
curl -X POST https://gateway.pipeworx.io/v1/tools/nd_statute \
  -H 'Content-Type: application/json' \
  -d '{"citation":"12.1-16-01"}'
```

No account needed for the first calls. Inspect any tool: `GET https://gateway.pipeworx.io/v1/tools/nd_statute`. Find one: `POST https://gateway.pipeworx.io/v1/tools/search_packs` with `{"query":"..."}`.

## Standalone (no gateway account)

This package also runs as a local stdio MCP server — no Pipeworx account, no
gateway round-trip:

```json
{
  "mcpServers": {
    "north-dakota-code": {
      "command": "npx",
      "args": ["-y", "@pipeworx/mcp-north-dakota-code"]
    }
  }
}
```

Or run it directly to confirm it starts:

```bash
npx -y @pipeworx/mcp-north-dakota-code
```

It speaks MCP over stdin/stdout and answers `initialize`/`tools/list`/`tools/call`
for **only** this pack's tools — none of the shared meta-tools the gateway
connection above adds. Same source, same tools, no ask_pipeworx routing.

## Using with ask_pipeworx

Instead of calling tools directly, you can ask questions in plain English —
this works on the pack endpoint above as well as on the full gateway:

```
ask_pipeworx({ question: "your question about North Dakota Code data" })
```

The gateway picks the right tool and fills the arguments automatically.

## More

- [Docs and guides](https://pipeworx.io/docs)
- [pipeworx.io](https://pipeworx.io)

## License

MIT
