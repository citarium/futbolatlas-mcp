# Football Atlas — MCP server

> The Laws of the Game and world football's major competitions, every claim cited to primary sources

A **remote MCP server** over a curated knowledge graph. Every claim it
returns is bound to a registered source: the tools hand back claims *with*
their citations and a confidence value, so an agent can show its work
instead of asserting.

Nothing to install. It is a hosted streamable-HTTP endpoint:

```
https://futbolatlas.app/mcp
```

## Add it to a client

**Claude Code**

```bash
claude mcp add --transport http futbol-atlas https://futbolatlas.app/mcp
```

**Claude Desktop / any client reading `mcpServers`**

```json
{
  "mcpServers": {
    "futbol-atlas": {
      "type": "streamable-http",
      "url": "https://futbolatlas.app/mcp"
    }
  }
}
```

No API key, no account, no auth. Read-only.

**Check it answers, without any client at all:**

```bash
curl -s https://futbolatlas.app/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'mcp-protocol-version: 2025-06-18' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

## Tools

Eight, each with an `outputSchema`, each returning `structuredContent`.

| tool | arguments | what it does |
|---|---|---|
| `get_overview` | — | Corpus overview: what this instance knows, counts by type, published tags, freshness. **Start here** when you land and do not yet know whether this corpus can answer your question. |
| `search` | `query`, `limit?` | Full-text search over the knowledge graph. Accent- and apostrophe-insensitive, so query in the user's own words; every hit carries its relevance score and the fields it matched. |
| `answer` | `question` | Answer a question from the corpus. Returns the matched object's claims with sources and confidence — **never an unsourced answer**. |
| `get_entity` | `id` | Fetch one knowledge object by id, with its claims and the sources each claim cites. |
| `get_topic` | `tag` | List the knowledge objects carrying a tag (topics are content-backed tags). |
| `get_related` | `id` | Graph neighbours of an object: outgoing and incoming relations, each with its relation type. |
| `get_sources` | `object_id?` | The whole source registry, or just the sources cited by one object. Use it to judge the corpus before trusting it. |
| `get_latest` | `limit?` | Most recently verified knowledge objects — a freshness signal. |

The intended path is `get_overview` → `search` or `answer` → `get_entity`
→ `get_related`. `get_overview` exists because an agent that has just
arrived needs to know whether this corpus can help *before* it spends a
call guessing.

## What is in the corpus

| | |
|---|---|
| knowledge objects | **141** |
| registered sources | **132** |
| published topics | **96** |

| type | objects |
|---|---|
| entity | 114 |
| glossary | 12 |
| faq | 7 |
| guide | 5 |
| comparison | 3 |

Subject matter: the seventeen Laws of the Game as the IFAB writes them, VAR protocol and its review categories, competition formats and qualification paths, offside, handball and the wording that decides them.

### Questions it is built to answer

- *What exactly does Law 11 say about a player's position at the moment the ball is played?*
- *When may VAR intervene, and when is it forbidden to?*
- *How does a team qualify for the Champions League league phase?*

## What an answer actually looks like

A real call against the live endpoint — `answer` with
*"when can VAR intervene"* — returns this `structuredContent`, trimmed:

```json
{
  "answered": true,
  "entity": {
    "id": "var",
    "name": "VAR (Video Assistant Referee)",
    "evidence_tier": "primary",
    "confidence": 0.94,
    "last_verified": "2026-08-08",
    "canonical_url": "https://futbolatlas.app/k/var"
  },
  "claims": [
    {
      "text": "The VAR may assist the referee only for a clear and obvious error or a serious missed incident in these match-changing categories: goal/no goal, penalty/no penalty, direct red card, and mistaken identity when the referee cautions or sends off the wrong player of the offending team.",
      "sources": [{ "title": "Video Assistant Referee (VAR) protocol (Laws of the Game 2026/27)" }]
    }
  ]
}
```

Note what travels with the answer: the **evidence tier**, a **confidence**,
the date it was **last verified**, and the **source behind the claim** — not
as prose an agent has to parse, but as fields it can act on. An agent can
decline to use a weak claim, or cite the primary source directly.

When the corpus cannot answer, `answered` is `false`. It does not
improvise, and the miss is recorded so the gap can be filled.

## Machine-readable surfaces

The MCP endpoint is one of several. The same corpus is served as plain
files an agent can read directly:

| surface | what it is |
|---|---|
| [`/llms.txt`](https://futbolatlas.app/llms.txt) | the index, as `text/plain` |
| [`/llms-full.txt`](https://futbolatlas.app/llms-full.txt) | the whole corpus in one file |
| [`/ai-index.json`](https://futbolatlas.app/ai-index.json) | every surface this instance publishes, with its content type |
| [`/api/index.json`](https://futbolatlas.app/api/index.json) | one JSON document per knowledge object |
| [`/api/sources.json`](https://futbolatlas.app/api/sources.json) | the source registry, in full |
| [`/.well-known/mcp/server.json`](https://futbolatlas.app/.well-known/mcp/server.json) | this server's manifest |

Each knowledge object has a human page and a machine twin at the same id,
with a canonical URL that agrees across all of them.

## Behaviour worth knowing before you integrate

- **`POST` only.** Every other method answers `405` with an `Allow: POST, OPTIONS` header.
- **Rate limit:** 120 requests per minute per client, counted in a shared
  store, published on every response as `RateLimit-Limit`,
  `RateLimit-Remaining` and `RateLimit-Reset` (all three exposed via CORS).
  It fails **open**: if the store is unreachable the request is served.
- **Malformed input** gets a spec-correct JSON-RPC error — `-32700` for
  unparseable bodies, `-32602` for an unknown tool — never an HTML error page.
- **Request bodies are capped** and validated before transport.

## Provenance and licence

Knowledge content is **CC-BY-4.0**: use it, cite it. The source registry is
public precisely so a claim can be checked rather than trusted —
`get_sources` returns what any given claim rests on.

Claims carry an evidence tier and a `last_verified` date. Where the
evidence is weaker, the object says so rather than rounding up.

## How it is built

Compiled and served by [Citarium](https://github.com/citarium/citarium), an
open-source framework for turning a knowledge graph into a website, an API,
an MCP server and agent-readable files from a single source — under
external evaluation, with the guardians and the falsification record in the
open.

This repository is the server's public face: its manifest and its
documentation. The corpus itself lives at [futbolatlas.app](https://futbolatlas.app).
