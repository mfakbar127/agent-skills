---
name: openapi-owasp-hurl
description: Generate Hurl plain-HTTP test files for all 10 OWASP API Security Top 10 2023 categories. Each .hurl file uses [Captures] for token chaining and [Asserts] for security assertions. Spawns one subagent per category in parallel and writes to security_test_cases/. Step 4b in the openapi-security-test pipeline.
---

# OpenAPI OWASP Hurl

Generate one `.hurl` file per OWASP API Security Top 10 2023 category by spawning 10 parallel subagents. Each file is a self-contained Hurl test suite runnable with `hurl --variables-file vars.env security_test_cases/apiN_*.hurl`.

## Hurl Format Primer

Each `.hurl` file is a sequence of **entries**. An entry = one HTTP request + optional response section.

```hurl
# --- Setup: obtain auth token ---
POST {{base_url}}/auth/login
Content-Type: application/json
{
  "username": "{{username}}",
  "password": "{{password}}"
}

HTTP 200
[Captures]
token: jsonpath "$.token"

# --- Test: access own resource ---
GET {{base_url}}/users/{{username}}
Authorization: Bearer {{token}}

HTTP 200
[Asserts]
jsonpath "$.username" == "{{username}}"
```

**Key syntax rules:**
- Request: `METHOD URL` on one line, then headers (no section label), then optional body
- Response: `HTTP <status>` (or `HTTP *` to accept any status)
- `[Captures]` — extract values for later entries: `varname: jsonpath "$.field"`
- `[Asserts]` — assertions on the response: `jsonpath "$.field" == "value"`, `status == 401`, `header "Content-Type" contains "application/json"`, `body contains "error"`
- Variables: `{{varname}}` in URLs, headers, bodies — supplied via `--variables-file vars.env` or `--variable key=value`
- Comments: `#` to end of line
- Multiple entries in one file, separated by blank lines

**vars.env format** (key=value, no quotes):
```
base_url=http://localhost:8080
username=user1
password=pass1
admin_username=admin
admin_password=adminpass
```

## Step 1 — Gather Context

Read each of these files if they exist. If none exist, read the raw spec directly. If `openapi-test-context.md` is absent but the other two are present, proceed — treat credentials and IDs as unknown and use `{{variable}}` placeholders.

1. `openapi-summary.md` — endpoint list, auth scheme, base URL
2. `openapi-dependencies.md` — session setup sequence, which endpoints need which data
3. `openapi-test-context.md` — payload catalog, stack-aware boundary values, real credentials, known IDs, tech stack, environment info

From these files (or the raw spec), collect:
- `BASE_URL` — e.g. `http://localhost:8080`
- `API_TITLE` — e.g. `My API`
- `AUTH_SCHEME` — e.g. `Bearer JWT obtained from POST /auth/login`
- `LOGIN_PATH` — the endpoint that returns a token
- `REGISTER_PATH` — the endpoint to create test accounts (if any)
- `TOKEN_JSONPATH` — JSONPath to extract the token from login response (e.g. `$.token`)
- `ENDPOINT_LIST` — full table of METHOD + path + auth required + description + request fields
- `TEST_USERS` — e.g. `regular: user1/pass1, admin: admin/adminpass`
- `KNOWN_IDS` — e.g. `item_id: abc123` (leave empty if none known)

## Step 2 — Generate vars.env

Write `security_test_cases/vars.env` with all known variables:

```
base_url={BASE_URL}
username={regular username or "user1"}
password={regular password or "changeme"}
admin_username={admin username or "admin"}
admin_password={admin password or "changeme"}
```

Add any known resource IDs as additional lines. If values are unknown, use a placeholder like `FILL_ME_IN`.

## Step 3 — Create security_test_cases/ Directory

```bash
mkdir -p security_test_cases
```

## Step 4 — Spawn 10 Subagents in Parallel

Send ALL 10 Agent calls in a **single message** so they run in parallel. Each agent receives a self-contained prompt — do not reference external files; inline the relevant endpoint data directly into each prompt.

Read the file `references/owasp-categories.md` located in the same directory as this skill (`.claude/skills/openapi-owasp-hurl/references/owasp-categories.md`) for the per-category checklist before writing the prompts.

### Agent Call Pattern

For each category, call the Agent tool with:
- `description`: short label e.g. `"API1 BOLA/IDOR Hurl tests"`
- `prompt`: the full self-contained prompt below

---

### API1 — Broken Object Level Authorization (BOLA/IDOR)

```
You are a security test engineer writing Hurl HTTP test files (https://hurl.dev).
Category: OWASP API Security Top 10 2023 — API1: Broken Object Level Authorization (BOLA/IDOR)
Risk: APIs tend to expose endpoints that handle object identifiers, creating a wide attack surface of Object Level Access Control issues. Object level authorization checks should be considered in every function that accesses a data source using an ID from the user.

API: {API_TITLE}
Base URL: {{base_url}}  (variable — set in vars.env)
Auth: {AUTH_SCHEME}
Login path: {LOGIN_PATH}
Token JSONPath: {TOKEN_JSONPATH}
Test users: {TEST_USERS}

Relevant endpoints (user-owned resources, path/query params identifying a resource):
{ENDPOINT_LIST filtered to endpoints with path/query params that identify a user-owned resource}

Hurl format rules:
- Each entry: METHOD URL, then headers (no label), then optional body, then HTTP <status>, then [Captures] and/or [Asserts]
- Use {{base_url}}, {{token}}, {{admin_token}}, {{username}}, {{other_username}} as variables
- Capture token from login: token: jsonpath "{TOKEN_JSONPATH}"
- Use `HTTP *` when asserting on the body regardless of status code
- Add a comment above each entry describing what it tests
- Group related entries (setup → attack → assert) with comments

Checklist — write at least one Hurl entry per item:
- Setup: login as user A, capture token_a; login as user B, capture token_b
- Access own resource with token_a (baseline — expect 200)
- Access user B's resource using token_a by manipulating path/query param (horizontal escalation — expect 403 or verify response does NOT contain user B's data)
- Access admin's resource using regular user token (vertical escalation — expect 403)
- Access protected resource without any token (expect 401)
- Access user B's resource with token_a and verify response body does not contain user B's private fields

Write a complete Hurl file. Then write it to:
security_test_cases/api1_bola_idor.hurl
Output ONLY the .hurl file content. No explanation.
```

---

### API2 — Broken Authentication

```
You are a security test engineer writing Hurl HTTP test files (https://hurl.dev).
Category: OWASP API Security Top 10 2023 — API2: Broken Authentication
Risk: Authentication mechanisms are often implemented incorrectly, allowing attackers to compromise authentication tokens or to exploit implementation flaws to assume other user's identities temporarily or permanently. Compromising a system's ability to identify the client/user, compromises API security overall.

API: {API_TITLE}
Base URL: {{base_url}}
Auth: {AUTH_SCHEME}
Login path: {LOGIN_PATH}
Register path: {REGISTER_PATH}
Token JSONPath: {TOKEN_JSONPATH}
Test users: {TEST_USERS}

Relevant endpoints: {ENDPOINT_LIST filtered to login, register, logout, and all auth-required endpoints}

Hurl format rules:
- Each entry: METHOD URL, headers, body, HTTP <status>, [Captures], [Asserts]
- Use HTTP * when you only want to assert on body/headers regardless of status
- Use `HTTP 401` to assert a specific rejection
- Repeat an entry N times by adding [Options] with repeat: N for rate-limit tests

Checklist — write at least one Hurl entry per item:
- Successful login returns token (baseline — HTTP 200, capture token)
- Login with wrong password → HTTP 401
- Login with non-existent user → HTTP 401, response body same as wrong password (no enumeration)
- Access protected endpoint without Authorization header → HTTP 401
- Access with malformed Authorization header (e.g. "Bearer bad.token.here") → HTTP 401
- Access with JWT signed by wrong secret (forge a token) → HTTP 401
- JWT alg:none attack — send token with alg=none in header → HTTP 401
- Brute-force: 20 rapid login attempts with wrong password (use repeat: 20), assert no 429 received — or assert 429 IS received (rate limiting present)
- Weak password at registration ("a") → assert accepted (vulnerability) or rejected (secure)
- After logout, reuse the old token on a protected endpoint → assert 401 (secure) or 200 (vulnerability)

Note: Apply JWT-specific checks (alg:none, wrong secret) only when {AUTH_SCHEME} uses JWT.
For session cookie auth, substitute: cookie theft via missing Secure/HttpOnly flags, session fixation.
For API key auth, substitute: key sent over HTTP, unauthenticated key discovery endpoint.

Write a complete Hurl file. Then write it to:
security_test_cases/api2_broken_authentication.hurl
Output ONLY the .hurl file content. No explanation.
```

---

### API3 — Broken Object Property Level Authorization

```
You are a security test engineer writing Hurl HTTP test files (https://hurl.dev).
Category: OWASP API Security Top 10 2023 — API3: Broken Object Property Level Authorization (Mass Assignment / Excessive Data Exposure)
Risk: This category combines API3:2019 Excessive Data Exposure and API6:2019 Mass Assignment, focusing on the root cause: the lack of or improper authorization validation at the object property level. This leads to information exposure or manipulation by unauthorized parties.

API: {API_TITLE}
Base URL: {{base_url}}
Auth: {AUTH_SCHEME}
Login path: {LOGIN_PATH}
Token JSONPath: {TOKEN_JSONPATH}
Test users: {TEST_USERS}

Relevant endpoints: {ENDPOINT_LIST filtered to endpoints that accept or return user object properties}

Checklist — write at least one Hurl entry per item:
- Setup: register and login, capture token
- Mass assignment via registration — send extra fields (role, isAdmin, score) in body; after login assert role/score was NOT elevated
- Mass assignment via profile update — PATCH/PUT with extra privileged fields; assert they were not applied
- Set a privileged field to high value at registration; re-fetch profile and assert value was not stored
- Excessive data exposure — GET profile/user endpoint; assert response body does NOT contain sensitive field names (passwordHash, __v, internalId) using `body not contains`
- Another user's property — login as user B, attempt to modify user A's object via mass assignment

Write a complete Hurl file. Then write it to:
security_test_cases/api3_broken_object_property_level_authorization.hurl
Output ONLY the .hurl file content. No explanation.
```

---

### API4 — Unrestricted Resource Consumption

```
You are a security test engineer writing Hurl HTTP test files (https://hurl.dev).
Category: OWASP API Security Top 10 2023 — API4: Unrestricted Resource Consumption
Risk: Satisfying API requests requires resources such as network bandwidth, CPU, memory, and storage. Successful attacks can lead to Denial of Service or an increase of operational costs.

API: {API_TITLE}
Base URL: {{base_url}}
Auth: {AUTH_SCHEME}
Login path: {LOGIN_PATH}
Token JSONPath: {TOKEN_JSONPATH}
Test users: {TEST_USERS}

Relevant endpoints: {ENDPOINT_LIST filtered to endpoints that accept large input, file uploads, or trigger external requests}

Hurl format rules:
- Use [Options] repeat: N to send N identical requests in sequence and test for rate limiting
- Assert `status == 429` to confirm rate limiting exists, or `status not == 429` to document absence of rate limiting

Checklist — write at least one Hurl entry per item:
- Rapid login attempts (repeat: 50) — assert 429 is returned (secure) or document it is not (vulnerability)
- Rapid registration (repeat: 20) — assert 429 or document absence
- Submit a 10,000-character string as a text field — assert accepted (vulnerability) or 400/413 (secure)
- If file upload endpoint exists: upload attempt with oversized body — assert 413 or document acceptance
- Rapid requests to any URL-fetch endpoint (repeat: 30) — assert 429 or document absence

Write a complete Hurl file. Then write it to:
security_test_cases/api4_unrestricted_resource_consumption.hurl
Output ONLY the .hurl file content. No explanation.
```

---

### API5 — Broken Function Level Authorization (BFLA)

```
You are a security test engineer writing Hurl HTTP test files (https://hurl.dev).
Category: OWASP API Security Top 10 2023 — API5: Broken Function Level Authorization (BFLA)
Risk: Complex access control policies with different hierarchies, groups, and roles tend to lead to authorization flaws. Attackers can gain access to other users' resources and/or administrative functions.

API: {API_TITLE}
Base URL: {{base_url}}
Auth: {AUTH_SCHEME}
Login path: {LOGIN_PATH}
Token JSONPath: {TOKEN_JSONPATH}
Test users: {TEST_USERS}
Known IDs: {KNOWN_IDS}  ← leave empty if none known; enumerate via the endpoint during tests

Relevant endpoints: {ENDPOINT_LIST filtered to endpoints described as admin-only, privileged, or role-restricted}

Checklist — write at least one Hurl entry per item:
- Setup: login as regular user (capture regular_token); login as admin (capture admin_token)
- Regular user calls admin-only endpoint with regular_token → assert HTTP 403 (or document 200 = vulnerability)
- Admin calls same endpoint with admin_token → assert HTTP 200 (baseline)
- Regular user accesses admin endpoint without any token → assert HTTP 401
- HTTP method tampering: send GET to a POST-only admin endpoint → assert 405 or 403 (or document 200)
- Parameter injection: add isAdmin=true or role=admin to request body as regular user → assert privilege was NOT granted
- Enumerate resource IDs on admin endpoint as regular user — assert each returns 403

Write a complete Hurl file. Then write it to:
security_test_cases/api5_broken_function_level_authorization.hurl
Output ONLY the .hurl file content. No explanation.
```

---

### API6 — Unrestricted Access to Sensitive Business Flows

```
You are a security test engineer writing Hurl HTTP test files (https://hurl.dev).
Category: OWASP API Security Top 10 2023 — API6: Unrestricted Access to Sensitive Business Flows
Risk: APIs vulnerable to this risk expose a business flow without compensating for how the functionality could harm the business if used excessively in an automated manner.

API: {API_TITLE}
Base URL: {{base_url}}
Auth: {AUTH_SCHEME}
Login path: {LOGIN_PATH}
Token JSONPath: {TOKEN_JSONPATH}
Test users: {TEST_USERS}

Relevant endpoints: {ENDPOINT_LIST filtered to endpoints that represent a meaningful business action — submission, ordering, voting, posting}

Checklist — write at least one Hurl entry per item:
- Automate a key business action: use repeat: 20 to send 20 rapid requests — assert 429 (rate-limited) or document 200 repeated (vulnerability)
- Bulk account creation: repeat: 20 registrations — assert 429 or document acceptance
- Duplicate submission of the same action twice — assert idempotency (second attempt rejected) or document it is accepted (vulnerability)
- Out-of-order flow: skip prerequisite step and submit final step directly — assert rejected or document acceptance
- Brute-force a submission value: repeat: 30 guesses — assert 429 or lockout
- Flood a resource-creating endpoint: repeat: 50 — assert 429 or document no limit

Write a complete Hurl file. Then write it to:
security_test_cases/api6_unrestricted_business_flows.hurl
Output ONLY the .hurl file content. No explanation.
```

---

### API7 — Server Side Request Forgery (SSRF)

```
You are a security test engineer writing Hurl HTTP test files (https://hurl.dev).
Category: OWASP API Security Top 10 2023 — API7: Server Side Request Forgery (SSRF)
Risk: SSRF flaws occur when an API fetches a remote resource without validating the user-supplied URI, enabling an attacker to coerce the application to send crafted requests to unexpected destinations.

API: {API_TITLE}
Base URL: {{base_url}}
Auth: {AUTH_SCHEME}
Login path: {LOGIN_PATH}
Token JSONPath: {TOKEN_JSONPATH}
Test users: {TEST_USERS}

Relevant endpoints: {ENDPOINT_LIST filtered to endpoints that accept a URL parameter and make a server-side HTTP request}

Guard: If the filtered endpoint list is empty (no URL-fetch endpoints exist in this API), write a single Hurl entry tagged with a comment "# @not-applicable — no URL-fetch endpoints found" that sends a GET to {{base_url}}/ and asserts HTTP 200, then write the file and stop.

Checklist — write at least one Hurl entry per item. For each, assert the server either blocked it (4xx) or document the response body content (evidence of SSRF):
- Setup: login, capture token
- Baseline: fetch a benign external URL → assert HTTP 200
- Fetch http://127.0.0.1/ → assert 400/403 (blocked) or assert body contains localhost content (SSRF)
- Fetch http://0.0.0.0/ and http://[::1]/ → assert blocked
- Fetch private IP: http://10.0.0.1/ → assert blocked
- Fetch AWS metadata: http://169.254.169.254/latest/meta-data/ → assert blocked or assert body contains "ami-id" (SSRF critical)
- Fetch file:///etc/passwd → assert blocked or assert body contains "root:" (SSRF critical)
- Fetch dict://localhost:6379/INFO → assert blocked
- Fetch with IP obfuscation: decimal (http://2130706433/), hex (http://0x7f000001/) → assert blocked

Write a complete Hurl file. Then write it to:
security_test_cases/api7_ssrf.hurl
Output ONLY the .hurl file content. No explanation.
```

---

### API8 — Security Misconfiguration

```
You are a security test engineer writing Hurl HTTP test files (https://hurl.dev).
Category: OWASP API Security Top 10 2023 — API8: Security Misconfiguration
Risk: APIs and the systems supporting them contain complex configurations. Missing security best practices open the door to different types of attacks.

API: {API_TITLE}
Base URL: {{base_url}}
Auth: {AUTH_SCHEME}

All endpoints: {ENDPOINT_LIST — full list}

Checklist — write at least one Hurl entry per item:
- Send request with invalid body to trigger a 4xx/5xx → assert body does NOT contain stack trace keywords ("at ", "Error:", "Exception", file paths)
- Send request with hostile Origin header → assert Access-Control-Allow-Origin response header is NOT "*"
- Send OPTIONS request to any endpoint → assert 405 or 200 with no sensitive info
- GET /swagger.json or /openapi.json or /api-docs without auth → assert 401/403 (secure) or document 200 (exposed)
- Send Content-Type: text/xml body to a JSON endpoint → assert 400 (validates content type) or document 200 (no validation)
- Send request over HTTP (if base URL is HTTPS, substitute http://) → assert redirect to HTTPS (301/302) or document plain HTTP accepted
- Check HSTS header: any authenticated response → assert header "Strict-Transport-Security" exists
- Check missing security headers: any response → assert header "X-Content-Type-Options" exists, assert header "X-Frame-Options" exists
- GET a sensitive endpoint (profile, user data) → assert header "Cache-Control" contains "no-store" or "no-cache"

Write a complete Hurl file. Then write it to:
security_test_cases/api8_security_misconfiguration.hurl
Output ONLY the .hurl file content. No explanation.
```

---

### API9 — Improper Inventory Management

```
You are a security test engineer writing Hurl HTTP test files (https://hurl.dev).
Category: OWASP API Security Top 10 2023 — API9: Improper Inventory Management
Risk: APIs tend to expose more endpoints than traditional web applications. Improper inventory leads to deprecated API versions and exposed debug endpoints.

API: {API_TITLE}
Base URL: {{base_url}}
Auth: {AUTH_SCHEME}

All documented endpoints: {ENDPOINT_LIST — full list}

Checklist — write at least one Hurl entry per item. For undocumented probes, assert 404 (not exposed) or document 200/non-404 (exposed = vulnerability):
- GET /swagger.json, /openapi.json, /api-docs, /swagger-ui/ without auth → assert 401/403 (secure) or document 200
- Versioned paths: for each documented path, probe /v1/<path>, /v2/<path> → assert 404 or document unexpected 200
- Admin paths: GET /api/admin, /admin, /api/management → assert 401/403/404 or document exposure
- Debug/health endpoints: GET /health, /status, /ping, /metrics, /api/debug, /api/env → assert 401/403/404 or document 200
- Common forgotten paths: GET /.env, /api/backup, /api/config, /api/test → assert 404 or document exposure
- Undocumented HTTP methods: send DELETE or PUT to a GET-only endpoint → assert 405 or document acceptance
- GET /uploads/ or /static/ → assert 403/404 (no directory listing) or document listing

Write a complete Hurl file. Then write it to:
security_test_cases/api9_improper_inventory_management.hurl
Output ONLY the .hurl file content. No explanation.
```

---

### API10 — Unsafe Consumption of APIs

```
You are a security test engineer writing Hurl HTTP test files (https://hurl.dev).
Category: OWASP API Security Top 10 2023 — API10: Unsafe Consumption of APIs
Risk: Developers tend to trust data received from third-party APIs more than user input, adopting weaker security standards. Attackers go after integrated third-party services instead of the target API directly.

API: {API_TITLE}
Base URL: {{base_url}}
Auth: {AUTH_SCHEME}
Login path: {LOGIN_PATH}
Token JSONPath: {TOKEN_JSONPATH}
Test users: {TEST_USERS}

Relevant endpoints: {ENDPOINT_LIST filtered to endpoints that fetch a remote URL or accept externally-sourced content}

Guard: If the filtered endpoint list is empty (no external-fetch endpoints exist in this API), write a single Hurl entry with comment "# @not-applicable — no external-fetch endpoints found" that sends a GET to {{base_url}}/ and asserts HTTP 200, then write the file and stop.

Checklist — write at least one Hurl entry per item:
- Setup: login, capture token
- Submit URL that returns an SSTI payload ({{7*7}}) as content → re-fetch stored content, assert body does NOT contain "49" (stored SSTI executed)
- Submit URL that returns an XSS payload (<script>alert(1)</script>) → re-fetch, assert body does NOT contain unescaped script tag
- Submit URL that returns 1MB+ body → assert server accepted without crashing (200) but note absence of size limit
- Submit URL returning wrong Content-Type (e.g. image/jpeg for JSON endpoint) → assert server validates or notes acceptance
- Submit URL that redirects to http://169.254.169.254/ (SSRF via redirect) → assert blocked or document access
- Submit URL with SQLi payload in returned body → re-fetch stored data, assert payload is escaped in response

Write a complete Hurl file. Then write it to:
security_test_cases/api10_unsafe_consumption_of_apis.hurl
Output ONLY the .hurl file content. No explanation.
```

---

## Step 5 — Syntax Check

```bash
hurlfmt --check security_test_cases/*.hurl
```

Fix any reported errors, then re-run until all files pass before continuing.

## Step 6 — Confirm Results

After all 10 agents complete and all files pass the syntax check, list the files in `security_test_cases/` and print a summary table:

| File | Status |
|------|--------|
| security_test_cases/vars.env | written |
| security_test_cases/api1_bola_idor.hurl | written |
| ... | ... |

If any file is missing, re-run that single agent with the same prompt.

## Step 7 — Print Run Instructions

Tell the user:

```
To run all tests:
  hurl --variables-file security_test_cases/vars.env security_test_cases/*.hurl

To run one category:
  hurl --variables-file security_test_cases/vars.env security_test_cases/api1_bola_idor.hurl

To run with a report:
  hurl --variables-file security_test_cases/vars.env --report-html reports/ security_test_cases/*.hurl

Edit security_test_cases/vars.env to set your base_url and credentials before running.
```

## Notes

- Replace all `{PLACEHOLDER}` values with real data before sending each prompt.
- Each subagent is self-contained — inline endpoint data, don't reference files by path.
- `{{variable}}` is Hurl runtime syntax — never replace these; they resolve via vars.env.
- Each `.hurl` file that needs a token must start with a login entry that captures it.
- Next step: `openapi-runner-hurl`.
