---
name: openapi-finding-reporting
description: Generate a structured finding report from security test output. Reads Cucumber.js JSON (reports/cucumber-report.json) or Hurl findings and writes finding-report.md with OWASP categories, severity, evidence, and remediation. Step 6 in the openapi-security-test pipeline.
---

# OpenAPI Finding Report Generator

Reads security test output and writes a structured security finding report to `finding-report.md`.

## Input — Determine Test Format

First determine which test format was used:

1. **Cucumber.js JSON** — look for a `.json` file:
   - `reports/cucumber-report.json`
   - `test-results.json`
   - Any `.json` file the user points to
   - See `references/cucumber-json-schema.md` for parsing details

2. **Hurl output** — look for structured findings from `openapi-runner-hurl`:
   - Step 8 findings table printed to the console
   - Verbose output captured by the user
   - Ask the user to paste the hurl output if not already in context

If neither is available, tell the user to run the test suite first:

**Cucumber.js:**
```bash
BASE_URL=http://localhost:3000 npx cucumber-js --format json:reports/cucumber-report.json
```

**Hurl:**
```bash
hurl --variables-file security_test_cases/vars.env --test security_test_cases/*.hurl
```

## Severity Mapping

See `references/owasp-severity.md` for the OWASP API Top 10 → severity/CVSS mapping.

## Parsing — Cucumber.js JSON

Read the JSON file. See `references/cucumber-json-schema.md` for structure.

Extract:
- Every **failing scenario** (`status: "failed"`) — confirmed vulnerabilities
- Every scenario with a step containing `"VULNERABLE"` in its output — marked findings even if the scenario passed overall
- Every scenario tagged `@wip` — **skip**
- Scenarios with status `pending` or `skipped` — list separately as "not executed"

For each confirmed finding extract:
- Feature name → maps to OWASP category via filename prefix (`api1` through `api10`)
- Scenario name
- The failing step and its error/output
- HTTP method + path from step names (e.g. `I send a GET request to "/api/getNote"`)
- Response status code if captured in step output

## Parsing — Hurl Output

From the hurl findings table (Step 8 of `openapi-runner-hurl`) or verbose output, extract:
- OWASP category (from filename prefix or comment in the `.hurl` file)
- HTTP method + path that triggered the finding
- Actual response status vs. expected status
- Any response body excerpt captured in verbose output

## Grouping

Group all findings by OWASP API category (API1 through API10). Within each category, there may be multiple findings if several scenarios/entries failed.

Use the filename prefix to determine category:

| Prefix | OWASP Category |
|--------|----------------|
| api1   | API1 — Broken Object Level Authorization |
| api2   | API2 — Broken Authentication |
| api3   | API3 — Broken Object Property Level Auth |
| api4   | API4 — Unrestricted Resource Consumption |
| api5   | API5 — Broken Function Level Authorization |
| api6   | API6 — Unrestricted Business Flows |
| api7   | API7 — Server-Side Request Forgery |
| api8   | API8 — Security Misconfiguration |
| api9   | API9 — Improper Inventory Management |
| api10  | API10 — Unsafe Consumption of APIs |

## Report Structure

Write `finding-report.md` using the following structure:

```markdown
# Security Finding Report
**Target:** <base URL from env, vars.env, or test context>
**Test suite:** <Cucumber.js / Gherkin | Hurl>
**Generated:** <today's date>
**Total findings:** <count confirmed> confirmed, <count manual review> manual review

---

## Summary

| Category | Severity | Status |
|----------|----------|--------|
| API1 - BOLA/IDOR | High | CONFIRMED |
| ... | ... | ... |

---

## Findings

### [API1] Broken Object Level Authorization (BOLA/IDOR)
**Severity:** High
**CVSS Base Score:** 8.1
**Affected endpoint:** `GET /api/resource`

**Description:**
<one paragraph — what the vulnerability is, how it manifests in this API>

**Evidence:**

Request:
\```http
GET /api/resource?id=victim HTTP/1.1
Host: <target host>
Authorization: Bearer <attacker_token>
\```

Response:
\```json
{
  "status": "success",
  "data": "..."
}
\```

**Scenario / Test that confirmed this:**
`<scenario name or hurl file + entry>`

**Remediation:**
<1-3 concrete sentences — what code change fixes this, referencing the specific endpoint and parameter>

---
```

Repeat the finding block for each confirmed finding, ordered by severity: Critical → High → Medium → Low.

If a finding needs manual review (e.g. mass assignment where the response is 201 but DB state wasn't verified, or a Hurl assertion mismatch that could be either a finding or a test artifact):

```markdown
**Status:** MANUAL REVIEW REQUIRED
**Reason:** <why automated confirmation wasn't possible>
```

### Not Executed / Skipped Section

At the end, add:

```markdown
## Not Executed

| Scenario / Test | Reason |
|-----------------|--------|
| ... | pending / skipped / requires manual setup |
```

## Evidence Notes

- **Cucumber.js**: response body is not captured by default. If no body is available, note "response body not captured" and suggest the user add `this.attach(JSON.stringify(this.response.data))` to `world.js` for richer evidence in future runs.
- **Hurl**: use the verbose output (`--verbose`) to capture full request/response pairs. If verbose was not run, note which entries need a re-run with verbose output.

## Output

Write the complete report to `finding-report.md` in the project root (or directory the user specifies).

Confirm the file was written and print the finding count:
```
finding-report.md written.
X confirmed, Y manual review, Z not executed.
```
