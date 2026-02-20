---
name: api2cli
description: Generate a working CLI from any API. Point it at API docs, a live URL, or a peek-api capture and get a dual-mode Commander.js CLI (human + agent output). Use when user wants to wrap an API in a CLI, generate a CLI from API docs, turn an API into a command-line tool, or scaffold a CLI from discovered endpoints.
---

# api2cli

Generate a working Node.js CLI from any API. Discovers endpoints, then scaffolds a dual-mode Commander.js CLI with a full-featured API client.

## Workflow

1. **Identify the API** -- user provides a docs URL, a live API base URL, or a peek-api capture
2. **Discover endpoints** -- parse docs, probe the API, or read a peek-api catalog
3. **Build endpoint catalog** -- normalize all discovered endpoints into a standard format
4. **Generate CLI** -- scaffold Commander.js CLI from the catalog
5. **User chooses destination** -- scaffold into current project or create standalone project

## Step 1: Identify the API

Ask the user:
- "What API do you want to wrap? Share a docs URL, a base URL, or point me at a peek-api capture."

Determine which discovery paths to use based on what they provide:

| Input | Discovery Path |
|-------|---------------|
| Docs URL (e.g., `https://docs.stripe.com/api`) | Docs parsing + active probing |
| Base URL (e.g., `https://api.example.com/v1`) | Active probing |
| peek-api capture dir (e.g., `./peek-api-linkedin/`) | Read existing catalog |
| Live website URL | Suggest running peek-api first, then active probing |

Also ask:
- "What auth does this API use?" (API key, Bearer token, cookies, OAuth, none)
- "Do you want this CLI in your current project or as a standalone project?"

## Step 2: Discover Endpoints

Use all applicable discovery paths. Combine results into a single catalog.

### Path A: Docs Parsing

1. Fetch the docs URL with WebFetch
2. Extract endpoint information: method, path, description, parameters, request/response examples
3. Look for pagination patterns, auth requirements, rate limit info
4. Follow links to sub-pages for individual endpoint docs if the main page is an index

### Path B: Active Probing

1. Check well-known paths for API specs:
   - `/.well-known/openapi.json`, `/.well-known/openapi.yaml`
   - `/openapi.json`, `/openapi.yaml`, `/swagger.json`, `/swagger.yaml`
   - `/api-docs`, `/docs`, `/api/docs`
   - `/graphql` (with introspection query)
2. Try `OPTIONS` on the base URL and common resource paths
3. Probe common REST patterns: `/api/v1/`, `/api/v2/`, `/v1/`, `/v2/`
4. For each discovered resource, try standard CRUD: `GET /resources`, `GET /resources/:id`, `POST /resources`, etc.
5. Parse response shapes to understand data models
6. Check response headers for rate limit info (`X-RateLimit-*`, `Retry-After`)
7. Check for pagination patterns in responses (`next`, `cursor`, `page`, `offset`)

See `references/discovery-strategies.md` for detailed probing patterns.

### Path C: peek-api Capture

1. Read the capture directory: `endpoints.json`, `auth.json`, `CAPTURE.md`
2. Parse endpoints into the standard catalog format
3. Extract auth headers and cookies from `auth.json`

If peek-api is not installed or no capture exists, tell the user:
```
To capture endpoints from a live site, install peek-api:
  git clone https://github.com/alexknowshtml/peek-api
  cd peek-api && npm install
  node bin/cli.js https://example.com
```

## Step 3: Build Endpoint Catalog

Normalize all discovered endpoints into this format:

```typescript
interface EndpointCatalog {
  service: string;           // e.g., "stripe", "nexudus"
  baseUrl: string;
  auth: {
    type: 'api-key' | 'bearer' | 'cookies' | 'oauth' | 'none';
    headerName?: string;     // e.g., "Authorization", "X-API-Key"
    envVar: string;          // e.g., "STRIPE_API_KEY"
  };
  pagination?: {
    style: 'cursor' | 'offset' | 'page' | 'link-header';
    paramName: string;       // e.g., "starting_after", "offset", "page"
    responseField: string;   // e.g., "has_more", "next", "next_page_url"
  };
  rateLimit?: {
    requests: number;
    window: string;          // e.g., "1m", "1h"
  };
  resources: ResourceGroup[];
}

interface ResourceGroup {
  name: string;              // e.g., "customers", "invoices"
  description: string;
  endpoints: Endpoint[];
}

interface Endpoint {
  method: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
  path: string;              // e.g., "/v1/customers/:id"
  description: string;
  parameters: Parameter[];
  requestBody?: object;      // JSON schema or example
  responseExample?: object;
}

interface Parameter {
  name: string;
  in: 'path' | 'query' | 'header';
  required: boolean;
  type: string;
  description: string;
}
```

Present the catalog to the user for review before generating:
```
Found 24 endpoints across 5 resources:
  customers (6 endpoints): list, get, create, update, delete, search
  invoices (5 endpoints): list, get, create, send, void
  ...
Ready to generate the CLI?
```

## Step 4: Generate CLI

Generate a dual-mode CLI using Commander.js. The CLI auto-detects human vs agent output via `process.stdout.isTTY`.

### File Structure

**In-project scaffold:**
```
scripts/
  {service}.ts                    # Entry point with shebang
  {service}/
    lib/
      client.ts                   # API client (auth, pagination, retry, caching)
      envelope.ts                 # Agent JSON envelope helpers
    commands/
      {resource}.ts               # One file per resource group
```

**Standalone project:**
```
{service}-cli/
  package.json
  tsconfig.json
  bin/
    {service}.ts                  # Entry point with shebang
  src/
    lib/
      client.ts
      envelope.ts
    commands/
      {resource}.ts
```

### Code Generation Patterns

See these references for the patterns to apply during generation:

- `references/api-client-template.md` -- API client class with pagination, retry, rate limiting, caching
- `references/agent-first-patterns.md` -- JSON envelope, HATEOAS next_actions, context-safe output, error fix suggestions
- `references/commander-patterns.md` -- Commander.js subcommands, global options, interactive prompts, colored output

### Key Generation Rules

**Entry point (`{service}.ts`):**
- Shebang: `#!/usr/bin/env npx tsx`
- Self-documenting root command (no args → prints full command tree as JSON)
- Global options: `--json` (force JSON output), `--verbose`, `--config <path>`

**API client (`lib/client.ts`):**
- Constructor takes base URL + auth config
- Auth from env var (name based on `catalog.auth.envVar`)
- Built-in pagination matching the API's pattern
- Retry with exponential backoff for 5xx and 429 errors
- Rate limiting based on discovered limits
- Optional response caching

**Envelope helpers (`lib/envelope.ts`):**
```typescript
const isAgent = !process.stdout.isTTY;

function respond(command: string, result: any, nextActions: Action[] = []) {
  if (isAgent) {
    console.log(JSON.stringify({ ok: true, command, result, next_actions: nextActions }));
  } else {
    return result; // caller handles human rendering
  }
}

function respondError(command: string, message: string, code: string, fix: string, nextActions: Action[] = []) {
  if (isAgent) {
    console.log(JSON.stringify({ ok: false, command, error: { message, code }, fix, next_actions: nextActions }));
  } else {
    console.error(`Error: ${message}`);
    console.error(`Fix: ${fix}`);
  }
  process.exit(1);
}
```

**Command files (`commands/{resource}.ts`):**
- One file per resource group
- Each endpoint becomes a subcommand: `mycli customers list`, `mycli customers get <id>`
- `list` commands: support `--limit`, `--offset`/`--cursor`, `--status` (if filterable)
- `get` commands: take ID as argument
- `create`/`update` commands: accept `--data <json>` or individual `--field` flags
- Every command includes contextual `next_actions` for agent mode
- Errors include `fix` suggestions

**Standalone project extras:**
- `package.json` with `commander`, `tsx` as dependencies, `bin` field pointing to entry
- `tsconfig.json` for TypeScript
- `.env.example` with the required env var

## Step 5: Post-Generation

After generating, do the following:

1. **Verify the CLI runs:** Execute with no args, confirm the self-documenting root works
2. **Test one endpoint:** Pick a simple GET endpoint, run it, verify output
3. **Tell the user what to do next:**

```
CLI generated at scripts/{service}.ts

To use it:
  npx tsx scripts/{service}.ts                    # See all commands
  npx tsx scripts/{service}.ts customers list     # List customers
  {SERVICE}_API_KEY=xxx npx tsx scripts/{service}.ts customers get abc123

For agent consumption (pipe to get JSON):
  npx tsx scripts/{service}.ts customers list | cat
```

## Reference Files

- `references/discovery-strategies.md` -- Detailed probing patterns, well-known paths, GraphQL introspection, response parsing
- `references/api-client-template.md` -- Full API client class with pagination, retry, rate limiting, caching
- `references/agent-first-patterns.md` -- Agent JSON envelope, HATEOAS, context-safe output, error handling
- `references/commander-patterns.md` -- Commander.js subcommands, nested commands, interactive prompts, colored output, config files, testing
