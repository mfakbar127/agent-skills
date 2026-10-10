# OWASP API Top 10 2023 — Severity and CVSS Reference

Use this to assign severity and CVSS base scores to findings. Scores are representative baselines — adjust based on actual exploitability in the target environment.

| Category | Name | Default Severity | CVSS Base | Key factors |
|---|---|---|---|---|
| API1 | Broken Object Level Authorization | High | 8.1 | AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N |
| API2 | Broken Authentication | Critical | 9.1 | AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N |
| API3 | Broken Object Property Level Auth | Medium | 6.5 | AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:H/A:N |
| API4 | Unrestricted Resource Consumption | Medium | 5.3 | AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:L — raise to High if DoS is achievable |
| API5 | Broken Function Level Authorization | High | 8.1 | AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N |
| API6 | Unrestricted Business Flows | Medium | 5.8 | AV:N/AC:L/PR:N/UI:N/S:C/C:N/I:L/A:N |
| API7 | Server-Side Request Forgery | High | 8.6 | AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:N/A:N — raise to Critical if cloud metadata confirmed |
| API8 | Security Misconfiguration | Medium | 5.3 | AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N — varies widely by misconfiguration type |
| API9 | Improper Inventory Management | Low | 3.7 | AV:N/AC:H/PR:N/UI:N/S:U/C:L/I:N/A:N |
| API10 | Unsafe Consumption of APIs | High | 7.5 | AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N |

## Severity adjustment rules

**Raise to Critical if:**
- SSRF reaches cloud metadata endpoint and returns credentials
- Authentication bypass grants admin access
- NoSQL injection bypasses login entirely

**Lower by one level if:**
- Finding requires authenticated access (reduce exploitability)
- Target is on an internal network only
- Finding is informational (header missing, version disclosed)

## Feature file → OWASP mapping

| Feature file prefix | OWASP category |
|---|---|
| api1 | API1 — Broken Object Level Authorization |
| api2 | API2 — Broken Authentication |
| api3 | API3 — Broken Object Property Level Auth |
| api4 | API4 — Unrestricted Resource Consumption |
| api5 | API5 — Broken Function Level Authorization |
| api6 | API6 — Unrestricted Business Flows |
| api7 | API7 — Server-Side Request Forgery |
| api8 | API8 — Security Misconfiguration |
| api9 | API9 — Improper Inventory Management |
| api10 | API10 — Unsafe Consumption of APIs |

## Remediation guidance by category

Write remediation advice that is specific to the target API's tech stack and the affected endpoint. Use the guidance below as a starting point, then tailor it to the actual framework, database, and endpoint observed in the finding.

### API1 — BOLA/IDOR
Enforce object-level ownership checks server-side before returning or modifying any resource. Never rely on a client-supplied identifier (query parameter, path segment, request body field) as the sole authorization check. Verify that the authenticated user owns or has explicit permission to access the requested object.

### API2 — Broken Authentication
Enforce strict token validation: reject unsigned or weakly signed tokens (e.g. `alg: none`), verify expiry, and pin the expected algorithm. Rate-limit authentication endpoints. Sanitize all login input before passing it to the database or authentication library to prevent injection-based bypass.

### API3 — Broken Object Property Level Auth
Never pass raw request bodies directly to ORM constructors or update calls. Define an explicit allowlist of fields the caller is permitted to set. Strip any field not on the allowlist before the write reaches the database.

### API4 — Unrestricted Resource Consumption
Apply rate limiting to all mutating and expensive endpoints. Enforce request body size limits at the framework level. Set timeouts and response size caps on any outbound HTTP requests triggered by user input.

### API5 — BFLA
Enforce role and permission checks server-side on every privileged endpoint. Never derive authorization from a role or privilege field supplied in the request body or header. Implement a middleware layer that validates the authenticated user's role before the handler executes.

### API6 — Unrestricted Business Flows
Identify the legitimate happy path and add server-side guards that prevent skipping or repeating steps (e.g. redeeming a coupon twice, bypassing payment). Use server-side state to track flow progress rather than trusting client-supplied step indicators.

### API7 — SSRF
Validate all user-supplied URLs against an explicit allowlist of permitted schemes and domains before making outbound requests. Block private IP ranges (127.x, 10.x, 172.16–31.x, 192.168.x), link-local addresses (169.254.x), and cloud metadata endpoints. Do not rely solely on DNS resolution results — validate after resolution.

### API8 — Security Misconfiguration
Disable verbose error messages and stack traces in production. Remove or restrict debug/admin endpoints. Enforce HTTPS, set secure headers (CORS, CSP, HSTS), and remove default credentials. Apply the principle of least privilege to all service accounts and API keys.

### API9 — Improper Inventory Management
Maintain an up-to-date inventory of all API versions and endpoints. Decommission or gate old API versions behind authentication. Ensure security controls applied to the current version are also applied to all older versions still reachable.

### API10 — Unsafe Consumption of APIs
Validate and sanitize all data received from third-party APIs before using it in business logic or database operations. Do not grant third-party API responses implicit trust. Apply the same input validation rules you apply to end-user input.
