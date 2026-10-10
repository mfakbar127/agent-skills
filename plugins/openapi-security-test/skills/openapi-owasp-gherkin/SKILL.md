---
name: openapi-owasp-gherkin
description: Generate Gherkin BDD .feature files for all 10 OWASP API Security Top 10 2023 categories. Spawns one subagent per category in parallel and writes to security_test_cases/. Step 4a in the openapi-security-test pipeline.
---

# OpenAPI OWASP Gherkin

Generate one `.feature` file per OWASP API Security Top 10 2023 category by spawning 10 parallel subagents.

## Step 1 — Gather Context

Read each of these files if they exist. If none exist, read the raw spec directly. If `openapi-test-context.md` is absent but the other two are present, proceed — treat credentials and IDs as unknown and generate runtime-fetch steps where values would be needed.

1. `openapi-summary.md` — endpoint list, auth scheme, base URL
2. `openapi-dependencies.md` — session setup sequence, which endpoints need which data
3. `openapi-test-context.md` — payload catalog, stack-aware boundary values, real credentials, known IDs, tech stack, environment info

From these files (or the raw spec), collect:
- `BASE_URL` — e.g. `http://localhost:8080`
- `API_TITLE` — e.g. `My API`
- `AUTH_SCHEME` — e.g. `Bearer JWT obtained from POST /auth/login`
- `ENDPOINT_LIST` — full table of METHOD + path + auth required + description + request fields
- `TEST_USERS` — e.g. `regular: user1/pass1, admin: admin/adminpass`
- `KNOWN_IDS` — e.g. `item_id: abc123` (leave empty if none known)

## Step 2 — Create security_test_cases/ Directory

```bash
mkdir -p security_test_cases
```

## Step 3 — Spawn 10 Subagents in Parallel

Send ALL 10 Agent calls in a **single message** so they run in parallel. Each agent receives a self-contained prompt — do not reference external files; inline the relevant endpoint data directly into each prompt.

Read the file `references/owasp-categories.md` located in the same directory as this skill (`.claude/skills/openapi-owasp-gherkin/references/owasp-categories.md`) for the per-category checklist before writing the prompts.

### Agent Call Pattern

For each category, call the Agent tool with:
- `description`: short label e.g. `"API1 BOLA/IDOR Gherkin scenarios"`
- `prompt`: the full self-contained prompt below

---

### API1 — Broken Object Level Authorization (BOLA/IDOR)

```
You are a security test engineer writing Gherkin BDD scenarios.
Category: OWASP API Security Top 10 2023 — API1: Broken Object Level Authorization (BOLA/IDOR)
Risk: APIs tend to expose endpoints that handle object identifiers, creating a wide attack surface of Object Level Access Control issues. Object level authorization checks should be considered in every function that accesses a data source using an ID from the user.

API: {API_TITLE}
Base URL: {BASE_URL}
Auth: {AUTH_SCHEME}
Test users: {TEST_USERS}

Relevant endpoints (user-owned resources, path/query params identifying a resource):
{ENDPOINT_LIST filtered to endpoints with path/query params that identify a user-owned resource}

Checklist — write at least one scenario per item:
- Access own resource (baseline happy path)
- Access another user's resource by manipulating path/query param (horizontal escalation)
- Access admin's resource (vertical escalation)
- Unauthenticated access to protected resource
- Token from user A used to access user B's resource
- Response does NOT leak other users' data
- Chained IDOR — use object from one endpoint as param to another

Write a complete Gherkin .feature file. Then write it to:
security_test_cases/api1_bola_idor.feature
Output ONLY the .feature file content. No explanation.
```

---

### API2 — Broken Authentication

```
You are a security test engineer writing Gherkin BDD scenarios.
Category: OWASP API Security Top 10 2023 — API2: Broken Authentication
Risk: Authentication mechanisms are often implemented incorrectly, allowing attackers to compromise authentication tokens or to exploit implementation flaws to assume other user's identities temporarily or permanently. Compromising a system's ability to identify the client/user, compromises API security overall.

API: {API_TITLE}
Base URL: {BASE_URL}
Auth: {AUTH_SCHEME}
Test users: {TEST_USERS}

Relevant endpoints (authentication, registration, logout, all protected endpoints):
{ENDPOINT_LIST filtered to login, register, logout, and all auth-required endpoints}

Checklist — write at least one scenario per item:
- Successful login returns valid JWT (baseline)
- Login with wrong password -> 401
- Login with non-existent user -> 401 (same message, no user enumeration)
- Access protected endpoint without token -> 401
- Access with empty / malformed Authorization header -> 401
- Access with expired JWT -> 401
- Access with tampered JWT payload (change username to admin, no re-sign) -> 401
- Access with JWT signed by wrong secret -> 401
- JWT alg:none attack -> 401
- 50 rapid login attempts — no 429 (brute-force unthrottled)
- Weak password accepted at registration ("a", "123456")
- Token still valid after logout (no server-side invalidation)
- NoSQL injection in login body

Note: Apply JWT-specific checks (alg:none, tampered payload, wrong secret, weak secret) only when {AUTH_SCHEME} uses JWT.
For session cookie auth, substitute: session fixation, cookie theft, missing SameSite/Secure/HttpOnly flags.
For API key auth, substitute: key enumeration in response headers, key sent over HTTP, unauthenticated key discovery.

Write a complete Gherkin .feature file. Then write it to:
security_test_cases/api2_broken_authentication.feature
Output ONLY the .feature file content. No explanation.
```

---

### API3 — Broken Object Property Level Authorization

```
You are a security test engineer writing Gherkin BDD scenarios.
Category: OWASP API Security Top 10 2023 — API3: Broken Object Property Level Authorization (Mass Assignment / Excessive Data Exposure)
Risk: This category combines API3:2019 Excessive Data Exposure and API6:2019 Mass Assignment, focusing on the root cause: the lack of or improper authorization validation at the object property level. This leads to information exposure or manipulation by unauthorized parties.

API: {API_TITLE}
Base URL: {BASE_URL}
Auth: {AUTH_SCHEME}
Test users: {TEST_USERS}

Relevant endpoints (registration, profile read/update, user data endpoints):
{ENDPOINT_LIST filtered to endpoints that accept or return user object properties}

Checklist — write at least one scenario per item:
- Mass assignment via registration — send extra fields such as score, role, isAdmin
- Mass assignment via profile update — send extra fields such as score, isAdmin
- Set privileged field to elevated value at registration
- Excessive data exposure — response contains internal fields like passwordHash, __v, internal IDs
- Response filtering — sensitive fields absent from user-facing responses
- Another user's property modified via mass assignment

Write a complete Gherkin .feature file. Then write it to:
security_test_cases/api3_broken_object_property_level_authorization.feature
Output ONLY the .feature file content. No explanation.
```

---

### API4 — Unrestricted Resource Consumption

```
You are a security test engineer writing Gherkin BDD scenarios.
Category: OWASP API Security Top 10 2023 — API4: Unrestricted Resource Consumption
Risk: Satisfying API requests requires resources such as network bandwidth, CPU, memory, and storage. Other resources such as emails/SMS/phone calls or biometrics validation are made available by service providers via API integrations, and paid for per request. Successful attacks can lead to Denial of Service or an increase of operational costs.

API: {API_TITLE}
Base URL: {BASE_URL}
Auth: {AUTH_SCHEME}
Test users: {TEST_USERS}

Relevant endpoints (login, registration, file upload, message/content submission, external URL fetch):
{ENDPOINT_LIST filtered to endpoints that accept large input, file uploads, or trigger external requests}

Checklist — write at least one scenario per item:
- 100 rapid login attempts — no 429
- 50 rapid registration attempts — no 429
- Upload oversized file — no 413
- Submit large string payload — accepted without truncation
- 50 rapid requests to any external-URL-fetch endpoint — no 429
- Fetch URL returning very large response — no read-size limit
- Many concurrent submissions — all accepted
- Any character/size limit hinted in spec — verify enforcement

Write a complete Gherkin .feature file. Then write it to:
security_test_cases/api4_unrestricted_resource_consumption.feature
Output ONLY the .feature file content. No explanation.
```

---

### API5 — Broken Function Level Authorization (BFLA)

```
You are a security test engineer writing Gherkin BDD scenarios.
Category: OWASP API Security Top 10 2023 — API5: Broken Function Level Authorization (BFLA)
Risk: Complex access control policies with different hierarchies, groups, and roles, and an unclear separation between administrative and regular functions, tend to lead to authorization flaws. By exploiting these issues, attackers can gain access to other users' resources and/or administrative functions.

API: {API_TITLE}
Base URL: {BASE_URL}
Auth: {AUTH_SCHEME}
Test users: {TEST_USERS}
Known IDs: {KNOWN_IDS}  ← leave empty if none known; enumerate via the endpoint during tests

Relevant endpoints (admin-only or role-gated functions):
{ENDPOINT_LIST filtered to endpoints described as admin-only, privileged, or role-restricted}

Checklist — write at least one scenario per item:
- Regular user calls admin-only endpoint -> expect 403, may get 200 (vulnerability)
- Admin user calls same endpoint (baseline)
- Enumerate admin endpoint resource IDs as regular user
- Retrieve unreleased/hidden resources via privileged filter parameter
- HTTP method tampering — GET on POST-only endpoints
- Unauthenticated access to privileged endpoint -> 401
- Parameter injection — add isAdmin:true or role:"admin" to body
- JWT role claim manipulation — forge token with elevated role
- Chained: enumerate hidden resource then use data in further attack

Write a complete Gherkin .feature file. Then write it to:
security_test_cases/api5_broken_function_level_authorization.feature
Output ONLY the .feature file content. No explanation.
```

---

### API6 — Unrestricted Access to Sensitive Business Flows

```
You are a security test engineer writing Gherkin BDD scenarios.
Category: OWASP API Security Top 10 2023 — API6: Unrestricted Access to Sensitive Business Flows
Risk: APIs vulnerable to this risk expose a business flow - such as buying a ticket, or posting a comment - without compensating for how the functionality could harm the business if used excessively in an automated manner. This doesn't necessarily come from implementation bugs.

API: {API_TITLE}
Base URL: {BASE_URL}
Auth: {AUTH_SCHEME}
Test users: {TEST_USERS}

Relevant endpoints (business flows: submission, scoring, registration, bulk operations):
{ENDPOINT_LIST filtered to endpoints that represent a meaningful business action — submission, scoring, ordering, voting, posting}

Checklist — write at least one scenario per item:
- Automate a key business action (e.g. submission) — many requests in short time, no rate limit
- Bulk account creation — many registrations, no throttle
- Duplicate submission — no re-award or idempotency protection
- Out-of-order flow — skip prerequisites and submit final step directly
- Brute-force a submission value — many guesses, no lockout
- Flood a resource-creating endpoint — many items from one user, no limit
- Automated scoreboard or ranking manipulation

Write a complete Gherkin .feature file. Then write it to:
security_test_cases/api6_unrestricted_business_flows.feature
Output ONLY the .feature file content. No explanation.
```

---

### API7 — Server Side Request Forgery (SSRF)

```
You are a security test engineer writing Gherkin BDD scenarios.
Category: OWASP API Security Top 10 2023 — API7: Server Side Request Forgery (SSRF)
Risk: Server-Side Request Forgery (SSRF) flaws can occur when an API is fetching a remote resource without validating the user-supplied URI. This enables an attacker to coerce the application to send a crafted request to an unexpected destination, even when protected by a firewall or a VPN.

API: {API_TITLE}
Base URL: {BASE_URL}
Auth: {AUTH_SCHEME}
Test users: {TEST_USERS}

Relevant endpoints (accepts a user-supplied URL and fetches it server-side):
{ENDPOINT_LIST filtered to endpoints that accept a URL parameter and make a server-side HTTP request}

Guard: If the filtered endpoint list is empty (no URL-fetch endpoints exist in this API), write a single Gherkin scenario tagged @not-applicable documenting that SSRF is out of scope for this API, then write the file and stop.

Checklist — write at least one scenario per item:
- Normal external URL fetch (baseline)
- Fetch http://127.0.0.1/ (localhost)
- Fetch http://0.0.0.0, [::1], decimal IP 2130706433
- Fetch private IP ranges: 10.x.x.x, 192.168.x.x, 172.16-31.x.x
- Fetch AWS metadata: http://169.254.169.254/latest/meta-data/
- Fetch AWS IAM credentials path
- Fetch GCP metadata: http://metadata.google.internal/
- Fetch Azure IMDS
- file:// scheme — file:///etc/passwd
- dict:// scheme — Redis INFO
- gopher:// scheme — raw TCP injection
- Internal service ports: Redis 6379, MongoDB 27017
- Open redirect to internal URL
- IP obfuscation: hex, decimal, IPv6, 127.1

Write a complete Gherkin .feature file. Then write it to:
security_test_cases/api7_ssrf.feature
Output ONLY the .feature file content. No explanation.
```

---

### API8 — Security Misconfiguration

```
You are a security test engineer writing Gherkin BDD scenarios.
Category: OWASP API Security Top 10 2023 — API8: Security Misconfiguration
Risk: APIs and the systems supporting them typically contain complex configurations, meant to make the APIs more customizable. Software and DevOps engineers can miss these configurations, or don't follow security best practices when it comes to configuration, opening the door for different types of attacks.

API: {API_TITLE}
Base URL: {BASE_URL}
Auth: {AUTH_SCHEME}

All endpoints:
{ENDPOINT_LIST — full list}

Checklist — write at least one scenario per item:
- Verbose error messages / stack traces in 4xx/5xx responses
- CORS — send hostile Origin header, check Access-Control-Allow-Origin: *
- HTTP method restriction — OPTIONS/TRACE/DELETE on GET/POST-only endpoints
- API documentation (Swagger/OpenAPI) accessible without auth
- Default or hardcoded credentials (JWT secret, admin password)
- Missing HTTPS / HTTP allowed
- No HSTS header
- Cookie Secure and HttpOnly flags absent
- Missing X-Content-Type-Options, X-Frame-Options, CSP headers
- No Cache-Control on sensitive endpoints
- Content-Type not validated — send XML/text to JSON endpoint
- Debug output leaks in error responses

Write a complete Gherkin .feature file. Then write it to:
security_test_cases/api8_security_misconfiguration.feature
Output ONLY the .feature file content. No explanation.
```

---

### API9 — Improper Inventory Management

```
You are a security test engineer writing Gherkin BDD scenarios.
Category: OWASP API Security Top 10 2023 — API9: Improper Inventory Management
Risk: APIs tend to expose more endpoints than traditional web applications, making proper and updated documentation highly important. A proper inventory of hosts and deployed API versions also are important to mitigate issues such as deprecated API versions and exposed debug endpoints.

API: {API_TITLE}
Base URL: {BASE_URL}
Auth: {AUTH_SCHEME}

All documented endpoints:
{ENDPOINT_LIST — full list}

Checklist — write at least one scenario per item:
- API spec / Swagger UI accessible without auth
- Versioned endpoints: /v1/, /v2/, /api/v1/ variants of every documented path
- Admin endpoints: /api/admin, /admin, /api/management
- Debug/test endpoints: /health, /status, /ping, /metrics, /api/debug, /api/env
- Abandoned or legacy endpoints hinted in spec descriptions
- Common forgotten paths: /api/backup, /api/config, /.env, /api/test
- Undocumented DELETE/PUT routes
- Static file directory listing
- Deprecated query parameters still accepted
- Error messages reveal internal routes or framework version

Write a complete Gherkin .feature file. Then write it to:
security_test_cases/api9_improper_inventory_management.feature
Output ONLY the .feature file content. No explanation.
```

---

### API10 — Unsafe Consumption of APIs

```
You are a security test engineer writing Gherkin BDD scenarios.
Category: OWASP API Security Top 10 2023 — API10: Unsafe Consumption of APIs
Risk: Developers tend to trust data received from third-party APIs more than user input, and so tend to adopt weaker security standards. In order to compromise APIs, attackers go after integrated third-party services instead of trying to compromise the target API directly.

API: {API_TITLE}
Base URL: {BASE_URL}
Auth: {AUTH_SCHEME}
Test users: {TEST_USERS}

Relevant endpoints (fetch or store content from external sources):
{ENDPOINT_LIST filtered to endpoints that fetch a remote URL or accept externally-sourced content}

Guard: If the filtered endpoint list is empty (no external-fetch endpoints exist in this API), write a single Gherkin scenario tagged @not-applicable documenting that Unsafe Consumption of APIs is out of scope, then write the file and stop.

Checklist — write at least one scenario per item:
- External URL returns SSTI payload — stored/executed server-side
- External URL returns stored XSS payload — reflected to other users
- External URL returns 1MB+ content — no read-size limit
- External URL returns binary/unexpected Content-Type — accepted without validation
- External URL redirects to internal resource (SSRF chain)
- External URL returns SQLi / NoSQLi payload — stored without sanitization
- External URL returns null bytes or special characters
- External URL has slow/hanging response — server hangs without timeout
- Content-Type mismatch between declared and actual response
- Trusting HTTP headers (X-Forwarded-For) from external response
- SVG/polyglot file via URL (XSS via SVG)

Write a complete Gherkin .feature file. Then write it to:
security_test_cases/api10_unsafe_consumption_of_apis.feature
Output ONLY the .feature file content. No explanation.
```

---

## Step 4 — Confirm Results

After all 10 agents complete, list the files in `security_test_cases/` and print a summary table:

| File | Status |
|------|--------|
| security_test_cases/api1_bola_idor.feature | written |
| ... | ... |

If any file is missing, re-run that single agent with the same prompt.

## Notes

- Replace all `{PLACEHOLDER}` values with real data before sending each prompt.
- Each subagent is self-contained — inline endpoint data, don't reference files by path.
- Next step: `openapi-test-runner`.
