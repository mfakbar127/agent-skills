# OWASP API Security Top 10 2023 — Per-Category Test Checklist

Use this as the subagent prompt appendix for each category. Each item = one scenario minimum.

Source: OWASP API Security Top 10 2023 (https://owasp.org/API-Security/editions/2023/en/0x11-t10.md)

---

## API1: Broken Object Level Authorization (BOLA / IDOR)

> APIs tend to expose endpoints that handle object identifiers, creating a wide attack surface of Object Level Access Control issues. Object level authorization checks should be considered in every function that accesses a data source using an ID from the user.

Relevant endpoint types: any endpoint with a user-owned resource, path params like `/{id}` or `/{username}`, query params that filter by owner.

Checklist:
- [ ] Access own resource (baseline happy path)
- [ ] Access another user's resource by manipulating path param (horizontal escalation)
- [ ] Access admin's resource by manipulating path param (vertical escalation)
- [ ] Access resource with numeric ID — try ID+1, ID-1
- [ ] Access resource with username param — try other known usernames
- [ ] Unauthenticated access to protected resource
- [ ] Token from user A used to access user B's resource
- [ ] Response does NOT leak other users' data in list endpoints
- [ ] Chained IDOR — use object from endpoint A as param to endpoint B

---

## API2: Broken Authentication

> Authentication mechanisms are often implemented incorrectly, allowing attackers to compromise authentication tokens or to exploit implementation flaws to assume other user's identities temporarily or permanently. Compromising a system's ability to identify the client/user, compromises API security overall.

Relevant endpoint types: login, register, logout, all JWT-protected endpoints.

Checklist:
- [ ] Successful login returns valid token (baseline)
- [ ] Login with wrong password → 401
- [ ] Login with non-existent user → 401 (same message as wrong password — no enumeration)
- [ ] Access protected endpoint without token → 401
- [ ] Access protected endpoint with empty/malformed Authorization header → 401
- [ ] Access with expired JWT → 401
- [ ] Access with tampered JWT payload (change username to admin, no signature update) → 401
- [ ] Access with JWT signed by wrong secret → 401
- [ ] JWT alg:none attack → 401
- [ ] Brute-force login — 50 rapid attempts, no rate limiting / 429
- [ ] Weak password accepted at registration (e.g., "a", "123456")
- [ ] Token still valid after logout (no server-side invalidation)
- [ ] NoSQL injection in login body (`{"$gt":""}`)

---

## API3: Broken Object Property Level Authorization

> This category combines API3:2019 Excessive Data Exposure and API6:2019 - Mass Assignment, focusing on the root cause: the lack of or improper authorization validation at the object property level. This leads to information exposure or manipulation by unauthorized parties.

Relevant endpoint types: register, profile update, any endpoint accepting JSON body with user attributes.

Checklist:
- [ ] Mass assignment via registration — send extra fields: `score`, `role`, `isAdmin`
- [ ] Mass assignment via profile update — send extra fields: `score`, `isAdmin`
- [ ] Attempt to set `score` to high value at registration
- [ ] Chained: mass-assign high score → check if hidden flag/reward unlocks
- [ ] Excessive data exposure — response contains `__v`, `passwordHash`, internal IDs
- [ ] Response filtering — sensitive fields absent from user-facing responses
- [ ] PUT/PATCH with extra body fields silently accepted
- [ ] Another user's property modified via mass assignment

---

## API4: Unrestricted Resource Consumption

> Satisfying API requests requires resources such as network bandwidth, CPU, memory, and storage. Other resources such as emails/SMS/phone calls or biometrics validation are made available by service providers via API integrations, and paid for per request. Successful attacks can lead to Denial of Service or an increase of operational costs.

Relevant endpoint types: login, register, file upload, note/message submission, external-URL-fetch endpoints.

Checklist:
- [ ] 100 rapid login attempts — no 429
- [ ] 50 rapid registration attempts — no 429
- [ ] Upload 50MB+ file — no 413 / no size limit
- [ ] Submit note with 1MB string — accepted without truncation
- [ ] Submit ticket with 500k-char message — accepted
- [ ] 50 rapid requests to external-URL-fetch endpoint — no 429
- [ ] Fetch URL returning very large response — no read-size limit
- [ ] 25 concurrent ticket submissions — all accepted
- [ ] Mixed concurrent requests across endpoints — no 503
- [ ] Character limit hint (if present in spec) — verify it is or is not enforced

---

## API5: Broken Function Level Authorization (BFLA)

> Complex access control policies with different hierarchies, groups, and roles, and an unclear separation between administrative and regular functions, tend to lead to authorization flaws. By exploiting these issues, attackers can gain access to other users' resources and/or administrative functions.

Relevant endpoint types: admin-only endpoints, role-gated endpoints, endpoints described as "only for administrators".

Checklist:
- [ ] Regular user calls admin-only endpoint → expect 403, may get 200 (vulnerability)
- [ ] Admin user calls same endpoint (baseline)
- [ ] Enumerate admin endpoint resource IDs as regular user
- [ ] Retrieve unreleased/hidden resources via `released=0` or similar filter
- [ ] HTTP method tampering — GET on POST-only, DELETE on GET-only
- [ ] Unauthenticated access to admin endpoint → 401
- [ ] Parameter injection — add `isAdmin: true` or `role: "admin"` to request body
- [ ] JWT role claim manipulation — forge token with elevated role
- [ ] Chained: enumerate hidden resource → use data from it in further attack

---

## API6: Unrestricted Access to Sensitive Business Flows

> APIs vulnerable to this risk expose a business flow - such as buying a ticket, or posting a comment - without compensating for how the functionality could harm the business if used excessively in an automated manner. This doesn't necessarily come from implementation bugs.

Relevant endpoint types: flag submission, scoring, bulk registration, ticket submission.

Checklist:
- [ ] Automated flag submission — 20 requests in 2 seconds, no rate limit
- [ ] Cycle through all challenge numbers in one burst
- [ ] Bulk account creation — 50 registrations, no throttle
- [ ] Duplicate flag submission — no re-award, but also check if rate-limited
- [ ] Out-of-order challenge solve — submit challenge 10 with 0 prior solves
- [ ] Flag brute-force — 50 guesses, no lockout or 429
- [ ] Ticket flooding — 100 tickets from one user, no limit
- [ ] Oversized ticket payload — 100k char message, stored without limit
- [ ] Scoreboard manipulation via rapid automated solves

---

## API7: Server Side Request Forgery (SSRF)

> Server-Side Request Forgery (SSRF) flaws can occur when an API is fetching a remote resource without validating the user-supplied URI. This enables an attacker to coerce the application to send a crafted request to an unexpected destination, even when protected by a firewall or a VPN.

Relevant endpoint types: any endpoint that accepts a URL and fetches remote content server-side.

Checklist:
- [ ] Normal external URL fetch (baseline)
- [ ] Fetch `http://127.0.0.1/` (localhost)
- [ ] Fetch `http://localhost:<port>/` variants (0.0.0.0, [::1], decimal IP)
- [ ] Fetch private IP ranges: 10.x.x.x, 192.168.x.x, 172.16–31.x.x
- [ ] Fetch AWS metadata: `http://169.254.169.254/latest/meta-data/`
- [ ] Fetch AWS IAM credentials path
- [ ] Fetch GCP metadata: `http://metadata.google.internal/computeMetadata/v1/`
- [ ] Fetch Azure IMDS: `http://169.254.169.254/metadata/instance`
- [ ] `file://` scheme — `file:///etc/passwd`
- [ ] `dict://` scheme — `dict://localhost:6379/INFO`
- [ ] `gopher://` scheme — raw TCP injection
- [ ] Internal service ports: Redis (6379), MongoDB (27017), Postgres (5432)
- [ ] Open redirect to internal — `http://attacker.com/redirect?to=http://169.254.169.254`
- [ ] IP obfuscation: hex, decimal, IPv6, `127.1`
- [ ] Character limit bypass — does truncation prevent full SSRF or just partial?

---

## API8: Security Misconfiguration

> APIs and the systems supporting them typically contain complex configurations, meant to make the APIs more customizable. Software and DevOps engineers can miss these configurations, or don't follow security best practices when it comes to configuration, opening the door for different types of attacks.

Relevant endpoint types: all endpoints; also check common paths.

Checklist:
- [ ] Verbose error messages / stack traces in 4xx/5xx responses
- [ ] CORS — send hostile `Origin` header, check `Access-Control-Allow-Origin: *`
- [ ] HTTP method restriction — OPTIONS/TRACE/DELETE on endpoints that only support GET/POST
- [ ] API documentation (Swagger/OpenAPI spec) accessible without authentication
- [ ] Default or hardcoded credentials (JWT secret, admin password)
- [ ] Missing HTTPS / HTTP allowed
- [ ] No HSTS header
- [ ] Cookie `Secure` and `HttpOnly` flags absent
- [ ] Missing `X-Content-Type-Options`, `X-Frame-Options`, `Content-Security-Policy`
- [ ] No `Cache-Control` on sensitive endpoints (profile, notes)
- [ ] Content-Type not validated — send XML/text to JSON endpoint
- [ ] Debug console.log output leaks in error responses

---

## API9: Improper Inventory Management

> APIs tend to expose more endpoints than traditional web applications, making proper and updated documentation highly important. A proper inventory of hosts and deployed API versions also are important to mitigate issues such as deprecated API versions and exposed debug endpoints.

Relevant endpoint types: all documented endpoints + probe for undocumented ones.

Checklist:
- [ ] API spec / Swagger UI accessible without auth
- [ ] Versioned endpoints: `/v1/`, `/v2/`, `/api/v1/` variants of every documented endpoint
- [ ] Admin endpoints: `/api/admin`, `/admin`, `/api/management`
- [ ] Debug/test endpoints: `/health`, `/status`, `/ping`, `/metrics`, `/api/debug`, `/api/env`
- [ ] Forgotten/abandoned endpoint from spec description hints (e.g., "developer left the company")
- [ ] Common forgotten paths: `/api/backup`, `/api/config`, `/.env`, `/api/test`
- [ ] Undocumented DELETE/PUT routes
- [ ] Static file directory listing (uploads folder)
- [ ] Deprecated query parameters still accepted
- [ ] Error messages reveal internal routes or framework name

---

## API10: Unsafe Consumption of APIs

> Developers tend to trust data received from third-party APIs more than user input, and so tend to adopt weaker security standards. In order to compromise APIs, attackers go after integrated third-party services instead of trying to compromise the target API directly.

Relevant endpoint types: any endpoint that fetches external content and stores or returns it.

Checklist:
- [ ] External URL returns SSTI payload — stored and executed server-side
- [ ] External URL returns stored XSS payload — reflected to other users
- [ ] External URL returns 1MB+ content — no size limit on read
- [ ] External URL returns binary/unexpected Content-Type — accepted without validation
- [ ] External URL redirects to internal resource (SSRF escalation chain)
- [ ] External URL returns SQLi / NoSQLi payload — stored without sanitization
- [ ] External URL returns null bytes or special characters
- [ ] External URL has slow response — server hangs without timeout
- [ ] Declared Content-Type vs actual Content-Type mismatch
- [ ] Trusting `X-Forwarded-For` or `X-Real-IP` headers from external response
- [ ] External SVG/polyglot file (XSS via SVG) uploaded via URL fetch
