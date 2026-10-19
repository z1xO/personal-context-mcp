personal-context-mcp — Doctolib tools, agent evals and Hermes Agent
A fork of PBenavides/personal-context-mcp, a TypeScript MCP server that gives AI agents access to personal tools (Google Calendar, YNAB, Hevy). This fork adds read-only Doctolib appointment search and an eval harness that scores how real LLMs use the tools, and connects the server to Hermes Agent.

· Upstream pull request: #9 (merged)

What I added
doctolib_search	Practitioners for a specialty in a city, with their open slots, soonest first
doctolib_practitioner	One profile: visit types, locations, booking link, and any visit type's slots for a date range
Eval harness (feat/agent-evals branch)	Runs any OpenRouter model against the server and scores tool choice, arguments, call count and answer with deterministic checks; native and prompted (Hermes <tool_call>) tool calling
Both Doctolib tools are read-only: they never book or cancel, and every result carries a link the user opens to book.

Results
6 endpoint-shaped tools redesigned into 2 intent-shaped ones: tool definitions 47% smaller, search responses 64% smaller.
Agents went from 2–3 tool calls per question to 0.9 (nemotron-3-super-120b, 6 cases × 2 runs).
Live testing found bugs the unit tests missed: 10 of 15 inherited specialty keywords returned nothing, and English is gb in Doctolib's filter, not en (0 vs 464 results).
Through Hermes Agent on claude-haiku-5.5, a real appointment question takes one tool call and costs under $0.01.
Run it
```bash
npm ci
cp .env.example .env # add Google Calendar keys to enable gcal_*
npm run dev # http://localhost:3000
```

The Doctolib tools need no credentials. They are served at POST /personal-tools. Check with curl localhost:3000/ready.

Use it from Hermes Agent
Add to Hermes's config.yaml (%LOCALAPPDATA%\hermes on Windows, ~/.hermes elsewhere):

```yaml
mcp_servers:
personal_tools:
url: "http://localhost:3000/personal-tools"
timeout: 120
```

Then hermes mcp test personal_tools and ask: find an English-speaking dentist in Berlin with a slot tomorrow.

Notes
Doctolib has no public API. The tools read the same public search and availability data as its website, anonymously and throttled, and do not get around its bot protection. From VPN or cloud IPs they return a doctolib.de link instead of results.
Built with Claude Code as an AI pair programmer.
Doctolib client ported from esinecan/doctolib-mcp.
The rest of the server is documented in [docs/](docs/).

# personal-context-mcp

A Fastify host that serves two MCP servers over **streamable HTTP**:

| Route | MCP | What it does |
|---|---|---|
| `POST /personal-context` | `personal-context` | Read Pablo's enriched personal context — Obsidian vault notes and activity events, classified by [`context-pipeline`](../context-pipeline/) into a 14-category taxonomy. Semantic search (S3 Vectors) with a BM25 fallback, plus an OWL ontology. |
| `POST /personal-tools` | `personal-tools` | Act on personal accounts — YNAB, Hevy and Google Calendar — and find doctor appointments on Doctolib (read-only). |

```
s3://…/context-enriched/ ──(loaded at boot, refreshed on new run_id)──► in-memory corpus
s3 vectors (Titan embeddings) ──────────────────────────────────────────► semantic search

client ──POST /personal-context (Bearer)──► search_context / get_document / list_taxonomy / ontology_*
client ──POST /personal-tools   (Bearer)──► ynab_* / hevy_* / gcal_* / doctolib_*  ──► YNAB, Hevy, Google Calendar, Doctolib
```

Production runs on a DigitalOcean droplet, reachable only over Tailscale.

## Quick start

You need **Node ≥ 20**. No credentials are needed for the first run.

```bash
npm ci
npm run dev
```

That serves the bundled test corpus (`tests/fixtures/enriched`) on
**http://localhost:3000** with auth turned off. Check it's up:

```bash
curl localhost:3000/health   # {"status":"ok"}
curl localhost:3000/         # lists the mounted MCPs
curl localhost:3000/ready    # which connections are configured / reachable
```

Without credentials, search runs in BM25 keyword mode, and the `ynab_*`,
`hevy_*` and `gcal_*` tools return a clear "not configured" error instead of
crashing.

## Running with real data and credentials

Copy the env template and fill in only what you need. `.env` is loaded
automatically and is gitignored. **Never commit it.**

```bash
cp .env.example .env
```

| To get… | Set in `.env` |
|---|---|
| Auth on the MCP routes (recommended even locally) | `MCP_BEARER_TOKEN` (any string locally) |
| The real corpus from S3 | `CORPUS_SOURCE=s3`, `CONTEXT_BUCKET=personal-pbenavides`, `AWS_PROFILE=personal` |
| Semantic search | `VECTOR_BUCKET=personal-context-vectors`, `VECTOR_INDEX=context-notes` (+ AWS creds with Bedrock and S3 Vectors access) |
| YNAB tools | `YNAB_TOKEN` |
| Hevy tools | `HEVY_API_KEY` |
| Google Calendar tools | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_CALENDAR_REFRESH_TOKEN` — see [docs/personal-tools.md](docs/personal-tools.md#secrets) for minting the token |
| Ontology injector + embeddings | `OPENROUTER_API_KEY` |

Then `npm run dev` as before. Each credential switches on its own tools; the
rest of the host is unaffected if one is missing.

Production values live in SSM under `/personal-context/*`. To read one:

```bash
aws ssm get-parameter --profile personal --region us-east-1 \
  --name /personal-context/mcp-bearer-token --with-decryption \
  --query Parameter.Value --output text
```

## Connecting a client

Claude Code, against your local server:

```bash
claude mcp add --transport http personal-context http://localhost:3000/personal-context \
  --header "Authorization: Bearer $MCP_BEARER_TOKEN"
claude mcp add --transport http personal-tools http://localhost:3000/personal-tools \
  --header "Authorization: Bearer $MCP_BEARER_TOKEN"
```

Drop the `--header` if you left `MCP_BEARER_TOKEN` empty. For production, swap
the URL for `http://100.118.218.44:8090/…` (needs Tailscale) and use the SSM
token.

## Debugging with the MCP Inspector

The [MCP Inspector](https://github.com/modelcontextprotocol/inspector) lets you
call tools by hand and see the raw JSON-RPC requests and responses. It's the
quickest way to check a tool's schema, its arguments, or why it returns an error.

### In the browser

With `npm run dev` running in another terminal:

```bash
npx @modelcontextprotocol/inspector
```

It prints a URL like `http://127.0.0.1:6274?MCP_INSPECTOR_API_TOKEN=…` and
opens it. In the UI:

1. **Transport:** Streamable HTTP
2. **URL:** `http://localhost:3000/personal-context` (or `/personal-tools`)
3. **Header:** `Authorization: Bearer <your MCP_BEARER_TOKEN>`, if you set one
4. **Connect**, then open **Tools → List Tools**, pick one, fill in the
   arguments and **Run Tool**

The history pane shows each request and response, which is where error
messages from the tool handlers appear.

### From the terminal (CLI mode)

Good for quick checks and scripting:

```bash
URL=http://localhost:3000/personal-context
AUTH="Authorization: Bearer $MCP_BEARER_TOKEN"

# list the tools and their input schemas
npx @modelcontextprotocol/inspector --cli $URL --transport http --header "$AUTH" \
  --method tools/list

# call one
npx @modelcontextprotocol/inspector --cli $URL --transport http --header "$AUTH" \
  --method tools/call --tool-name search_context --tool-arg query=knee
```

### Against production

Same commands, with the droplet's URL over Tailscale and the token from SSM:

```bash
URL=http://100.118.218.44:8090/personal-tools
AUTH="Authorization: Bearer $(aws ssm get-parameter --profile personal --region us-east-1 \
  --name /personal-context/mcp-bearer-token --with-decryption --query Parameter.Value --output text)"
```

Careful: production `personal-tools` writes go to the real YNAB, Hevy and
Google accounts, and several of those APIs can't undo a write. Pass
`dry_run: true` where a tool offers it.

### Other debugging tips

- **`/ready`** shows which connections are configured and reachable. Start
  there when a tool says "not configured".
- **`LOG_LEVEL=debug npm run dev`** for verbose logs.
- **`search_context` returns `mode`**: `semantic` or `keyword`. `keyword` when
  you expected semantic means the vector index isn't configured or the query
  to it failed (check the logs).
- **401** means the `Authorization` header is missing or doesn't match
  `MCP_BEARER_TOKEN`. **405** on a GET is expected: the transport is
  stateless and only accepts POST.

## Tests and checks

```bash
make verify   # typecheck + lint + unit tests + smoke test — the same gate CI runs
```

Individual steps: `npm run typecheck`, `npm run lint`, `npm test`,
`make smoke`. The smoke test (`scripts/mcp_smoke.sh`) boots the server on
:3999 against the fixtures and drives it over real JSON-RPC, so it needs no
credentials. Unit tests sit next to their source as `*.test.ts` and run
against fakes, never the network.

## Shipping a change

Nothing lands on `main` except through a PR, and **every merge to `main`
deploys to production automatically** (build → ECR → roll the droplet
container → smoke test → auto-rollback on failure). Run `make hooks` once per
clone to install the pre-push guard against pushing to `main` directly.

Details: [docs/CICD.md](docs/CICD.md) and [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md).

## Tools

### `personal-context`

| Tool | Purpose |
|---|---|
| `search_context` | Search notes and events, optionally filtered by `category`, `status` or `horizon`. Semantic when the vector index is configured, BM25 otherwise. |
| `get_document` | Full document (frontmatter + body) or a bounded preview. |
| `list_taxonomy` | Categories with document counts and taxonomy/manifest versions. |
| `ontology_search` | Search the OWL ontology graph, with neighbourhood expansion. |
| `ontology_update` | Schema-validated upsert into the ontology; saves a new immutable version. |
| `ontology_sync` | Run the ontology injector on demand (same job as the hourly cron). |

### `personal-tools`

| Family | Tools | Notes |
|---|---|---|
| YNAB | `ynab_overview`, `ynab_transactions`, `ynab_categories`, `ynab_report`, `ynab_lookup`, `ynab_assign`, `ynab_write_transaction`, `ynab_write_category` | Built around the one-month-ahead method. Amounts in currency units; every write takes `dry_run`. |
| Hevy | `hevy_recent_activity`, `hevy_get_workout`, `hevy_routines`, `hevy_body_measurements`, `hevy_calculate_burnt_calories`, `hevy_search_exercise_catalog`, `hevy_write_routine` | Read-only except `hevy_write_routine` (with `dry_run`). |
| Google Calendar | `gcal_list_calendars`, `gcal_list_events`, `gcal_get_event`, `gcal_create_event`, `gcal_update_event`, `gcal_delete_event`, `gcal_free_busy` | Defaults to the primary calendar. |
| Doctolib | `doctolib_search`, `doctolib_practitioner` | Read-only, no credentials. Search returns practitioners with their earliest slots; practitioner gives motives, places, a booking link, and a motive's slots. See [docs/doctolib.md](docs/doctolib.md). |

## Configuration reference

Config comes from `config/default.json`, overridden by env vars mapped in
`config/custom-environment-variables.json`. [.env.example](.env.example) lists
every variable with comments.

| Env | Default | Meaning |
|---|---|---|
| `PORT` | `3000` | HTTP port. |
| `LOG_LEVEL` | `info` | Pino log level. |
| `MCP_BEARER_TOKEN` | *(empty = no auth!)* | Bearer required on MCP routes. `/health`, `/ready` and `/` stay open. |
| `CORPUS_SOURCE` | `file` | `file` or `s3`. |
| `CORPUS_ROOT` | `tests/fixtures/enriched` | Directory for the `file` source. |
| `CONTEXT_BUCKET` / `CONTEXT_PREFIX` | — / `context-enriched/` | S3 location for the `s3` source. |
| `REFRESH_SECONDS` | `300` | How often to check the manifest for a new run; `0` disables. |
| `VECTOR_BUCKET` / `VECTOR_INDEX` | *(empty = BM25 only)* | S3 Vectors index for semantic search. |
| `EMBED_MODEL_ID` | `amazon.titan-embed-text-v2:0` | Bedrock model used to embed queries. |
| `AWS_REGION` | `us-east-1` | Region for S3, S3 Vectors and Bedrock. |
| `ONTOLOGY_SOURCE` / `ONTOLOGY_ROOT` / `ONTOLOGY_BUCKET` / `ONTOLOGY_PREFIX` | `file` / `data/ontology` / — / `ontology/` | Where the ontology registry lives. |
| `ONTOLOGY_SYNC_ENABLED` | `false` | Hourly injector cron. |
| `OPENROUTER_API_KEY` | *(empty = injector off)* | Secret. |
| `YNAB_TOKEN` | *(empty = `ynab_*` off)* | Secret. |
| `HEVY_API_KEY` | *(empty = `hevy_*` off)* | Secret. |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` / `GOOGLE_CALENDAR_REFRESH_TOKEN` | *(any empty = `gcal_*` off)* | Secrets. |
| `DOCTOLIB_COUNTRY` / `DOCTOLIB_MIN_INTERVAL_MS` | `de` / `600` | Default Doctolib site and per-host request throttle. Not secrets. |

All secrets come from `.env` locally and from SSM `/personal-context/*` in
production.

## Where to read next

| Doc | Covers |
|---|---|
| [AGENTS.md](AGENTS.md) | Directory map and coding conventions. |
| [docs/mcp-registry.md](docs/mcp-registry.md) | Adding a new MCP. |
| [docs/connections.md](docs/connections.md) | Adding an external dependency. |
| [docs/personal-context-mcp.md](docs/personal-context-mcp.md) | Corpus MCP: tools, search, corpus shape. |
| [docs/personal-tools.md](docs/personal-tools.md) | YNAB / Hevy / Calendar tools and their secrets. |
| [docs/doctolib.md](docs/doctolib.md) | The Doctolib tools: endpoints, API quirks, what wasn't ported. |
| [docs/ontologies.md](docs/ontologies.md) | The OWL ontology layer. |
| [docs/server-and-hosting.md](docs/server-and-hosting.md) | App assembly, config, auth, management routes. |
