# API Discovery Strategies

Detailed patterns for discovering API endpoints from live services.

## Well-Known Spec Paths

Check these paths first -- if any return an OpenAPI/Swagger spec, parse it directly:

```
/.well-known/openapi.json
/.well-known/openapi.yaml
/openapi.json
/openapi.yaml
/openapi/v3/api-docs
/swagger.json
/swagger.yaml
/swagger/v1/swagger.json
/api-docs
/api-docs.json
/docs/api
/api/docs
/api/swagger
/api/v1/swagger.json
/api/v2/swagger.json
```

If a spec is found, use it as the primary source and skip active probing for covered endpoints.

## Active Probing Patterns

### Base URL Discovery

Try common API base paths:

```
/api/
/api/v1/
/api/v2/
/api/v3/
/v1/
/v2/
/rest/
/graphql
```

For each, send a `GET` request and check:
- 200 with JSON body → likely an API root, inspect response for resource links
- 401/403 → API exists, needs auth
- 404 with JSON error body → API framework present, wrong path
- 404 with HTML → not an API path

### Resource Discovery

Once a base path is found, probe common resource names:

```
/users
/accounts
/customers
/orders
/products
/items
/messages
/notifications
/events
/projects
/tasks
/comments
/posts
/files
/uploads
/settings
/config
/health
/status
/me
/profile
```

For each resource that returns 200 or 401:
1. It exists -- add to catalog
2. Try `OPTIONS` to discover allowed methods
3. If list returns items, extract an ID and try `GET /resource/{id}`
4. Check response for nested resource links (e.g., `/users/123/orders`)

### CRUD Probing

For each confirmed resource, test standard operations:

```
GET    /resources           → List
GET    /resources/:id       → Get single
POST   /resources           → Create (send empty body, check error for required fields)
PUT    /resources/:id       → Full update
PATCH  /resources/:id       → Partial update
DELETE /resources/:id       → Delete
```

Also check common non-CRUD endpoints:

```
GET    /resources/search?q=test    → Search
GET    /resources/count            → Count
POST   /resources/:id/archive      → Action endpoints
POST   /resources/bulk             → Bulk operations
GET    /resources/:id/related      → Related resources
```

### Response Analysis

Parse responses to understand:

**Pagination style:**
```json
// Cursor-based
{ "data": [...], "has_more": true, "next_cursor": "abc123" }

// Offset-based
{ "data": [...], "total": 150, "offset": 0, "limit": 20 }

// Page-based
{ "data": [...], "page": 1, "total_pages": 8 }

// Link header
Link: <https://api.example.com/items?page=2>; rel="next"
```

**Auth requirements (from error responses):**
```json
// API key
{ "error": "Missing API key", "code": "authentication_required" }

// Bearer token
{ "error": "Invalid or expired token" }

// Check WWW-Authenticate header for auth scheme
WWW-Authenticate: Bearer realm="api"
```

**Rate limits (from response headers):**
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1620000000
Retry-After: 60
```

**Data models (from response bodies):**
Extract field names, types, and relationships from response objects to inform CLI flag generation.

## GraphQL Introspection

If `/graphql` returns 200, try the introspection query:

```graphql
{
  __schema {
    queryType { name }
    mutationType { name }
    types {
      name
      kind
      fields {
        name
        type { name kind ofType { name kind } }
        args { name type { name kind } }
      }
    }
  }
}
```

Map GraphQL types to CLI commands:
- Queries → `get` and `list` subcommands
- Mutations → `create`, `update`, `delete` subcommands
- Each query/mutation argument → CLI flag

## Docs Page Parsing

When working with human-readable docs pages:

1. Fetch the page with WebFetch
2. Look for structured patterns:
   - Endpoint tables (`GET /users` in table or code block format)
   - Request/response examples (JSON code blocks)
   - Parameter lists (tables or definition lists)
   - Auth sections (API key format, header names)
3. Follow navigation links to individual endpoint pages for full details
4. Extract: method, path, description, parameters (name, type, required), example request/response

Common docs frameworks to recognize:
- **Swagger UI**: Look for the spec URL in page source
- **ReadMe.io**: Structured endpoint blocks with try-it forms
- **Slate/Docusaurus**: Markdown-rendered endpoint docs
- **Postman docs**: Collection-based endpoint listing
- **Redoc**: OpenAPI-rendered docs, spec URL often in page source
