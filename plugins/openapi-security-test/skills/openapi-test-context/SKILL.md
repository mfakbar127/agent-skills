---
name: openapi-test-context
description: Collect tech stack info and runtime test data from the user, auto-extract payload fields, and generate stack-aware boundary values. Writes openapi-test-context.md for use by openapi-owasp-gherkin and openapi-owasp-hurl. Step 3 in the openapi-security-test pipeline.
---

# OpenAPI Test Context

Collect tech stack info and runtime data from the user, then enrich it with spec-extracted payload details and stack-aware boundary values. Write everything to `openapi-test-context.md`.

## Workflow

### Phase 1 — Read spec context

Read `openapi-summary.md` and `openapi-dependencies.md` if present. If absent, read the raw spec. Extract:
- All endpoints with method, path, request fields, response fields
- Auth scheme (JWT, session, API key, etc.)
- Base URL

### Phase 2 — Ask the user (one message, grouped)

Present all questions in a single message. Skip categories that don't apply to this spec.

#### Category 0 — Tech Stack (ask first — shapes all boundary values)
- Backend language and framework? (e.g. Node.js/Express, Python/Flask, Java/Spring)
- Database type? (e.g. MongoDB, MySQL, PostgreSQL, Redis, SQLite)
- Cloud provider / hosting? (AWS, GCP, Azure, on-prem, Docker local, unknown)
- Any known middleware? (rate limiter, WAF, API gateway, reverse proxy)

#### Category A — Test Environment
- What is the base URL of the running API? (default from spec: `<url>`)
- Is HTTPS enforced, or is HTTP also accepted?

#### Category B — User Accounts
- Do you have a **regular user** account for testing? username / password?
- Do you have an **admin user** account? username / password?
- Should tests create fresh accounts via the registration endpoint, or use pre-existing ones?
- Are there seeded/demo accounts already in the system?

#### Category C — Auth Tokens
(Only if spec uses JWT or token-based auth)
- Do you have a valid **auth token** for a regular user? (paste it, or leave blank)
- Do you have a valid **admin token**? (paste it, or leave blank)
- What is the token TTL / expiry?
- Is the JWT secret known? (leave blank if unknown)

#### Category D — Known Resource IDs
(Ask per resource type found in the spec)
- Known resource IDs that exist in the system? (e.g. user IDs, order IDs, item IDs — one per resource type found in the spec)
- Known object IDs or path parameter values to use in IDOR tests?

#### Category E — External Infrastructure
(Only if spec has an endpoint that fetches external URLs)
- Do you have a controlled external server for SSRF tests?
- Is a cloud metadata endpoint reachable from the server?
- What internal services run alongside the API? (e.g. Redis:6379, MongoDB:27017)

#### Category F — Constraints
- Endpoints to avoid during testing? (e.g. destructive ops, billing)
- Any rate limits already in place the tests should respect?
- Should the test suite clean up created accounts/data after running?

### Phase 3 — Auto-extract payload catalog from spec

For every endpoint with a requestBody or query/path parameters, extract:
- Required vs optional fields, types, example values
- Inferred constraints from spec (length hints, enum values)

Then generate boundary test values enriched by the user's stack answers:

| Stack answer | Additional boundary values to add |
|---|---|
| MongoDB | `{"field": {"$gt": ""}}`, `{"field": {"$regex": ".*"}}` |
| MySQL / PostgreSQL | `"' OR '1'='1"`, `"'; DROP TABLE users;--"` |
| AWS | SSRF to `http://169.254.169.254/latest/meta-data/` |
| GCP | SSRF to `http://metadata.google.internal/` |
| Azure | SSRF to `http://169.254.169.254/metadata/instance` |
| Redis (internal) | `dict://localhost:6379/INFO`, `gopher://localhost:6379/` |
| Node.js | `"__proto__[admin]=true"` (prototype pollution) |
| Python / Flask | SSTI: `{{7*7}}`, `{{config}}`, `{{''.__class__.__mro__}}` |
| Java / Spring | SpEL injection: `${7*7}`, `T(java.lang.Runtime).getRuntime().exec('id')` |
| JWT | alg:none, HS256/RS256 confusion, weak secret wordlist |
| File upload present | 0-byte, 50MB, wrong MIME, JPEG+PHP polyglot, SVG with XSS |

Always include these generic boundary values regardless of stack:
- Empty string `""`, `null`, very long string (10,000 chars), wrong type (number where string expected), path traversal `"../../../etc/passwd"`, unicode null byte `"\u0000"`

### Phase 4 — Write output

Write `openapi-test-context.md` with all collected and derived data.

## Output Format

```markdown
# Test Context

## Environment
- Base URL: <from spec servers[0].url>
- HTTPS enforced: <yes/no>
- Tech stack: <framework> / <db> / <auth scheme>
- Cloud provider: <aws/gcp/azure/on-prem/unknown>
- Middleware: <list or "none known">

## Endpoints & Payloads

### POST /auth/register
**Auth:** none
**Fields:** username (string, required), password (string, required)
**Baseline payload:** {<field1>: "<value1>", <field2>: "<value2>"}  ← fill from spec examples or field defaults
**Boundary values:**
- username: `""`, `null`, `"a"`, `"a".repeat(10000)`, `{"$gt":""}` (NoSQLi), `"' OR '1'='1"` (SQLi), `"__proto__[admin]=true"` (proto pollution)
- password: `""`, `null`, `"x"`, `"password"`, `"123456"`

### GET /users/{username}
**Auth:** bearerAuth
**Path params:** username (string, required)
**Boundary values:**
- username: own username, another user's username, `"admin"`, `"../../../etc"` (path traversal)

### POST /items/fetch
**Auth:** bearerAuth
**Fields:** url (string, required)
**Boundary values:**
- url: `"http://127.0.0.1/"`, `"http://169.254.169.254/latest/meta-data/"`, `"file:///etc/passwd"`, `"dict://localhost:6379/INFO"`, `"http://0.0.0.0/"`, `"http://[::1]/"`, private ranges (10.x, 192.168.x, 172.16-31.x)

---

## User Accounts
- Regular user: username=<user1>, password=<pass1>
- Admin user: username=<admin>, password=<adminpass> (or: unknown)
- Seeded accounts: <list if any, else "none">
- Strategy: create fresh accounts via registration endpoint

## Auth Tokens
- Regular user token: <token or "generate at runtime">
- Admin token: <token or "unknown">
- Token TTL: <ttl or "unknown">
- JWT secret: unknown (or known value if provided)

## Known Resource IDs
- <resource type>: <id value>
- (none known — will enumerate at runtime)

## External Infrastructure
- Controlled SSRF server: <url or "none">
- Cloud metadata reachable: unknown
- Internal services: <list or "none known">

## Constraints
- Endpoints to avoid: none
- Cleanup after tests: yes
- Known rate limits: none
```

## Notes

- Unanswered questions: record `unknown`, don't block.
- Next step: `openapi-owasp-gherkin` (Gherkin) or `openapi-owasp-hurl` (Hurl).
