---
name: openapi-runner-hurl
description: Run the .hurl files in security_test_cases/ against a live API, interpret results, and print a findings table. Step 5b in the openapi-security-test pipeline.
---

# OpenAPI Runner Hurl

Execute the `.hurl` files in `security_test_cases/` against a live API and report confirmed findings.

## Prerequisites

- `hurl` CLI installed (`hurl --version` to verify; install from https://hurl.dev/docs/installation.html)
- API server is running and reachable at `base_url` in `vars.env`
- `security_test_cases/vars.env` is populated (no `FILL_ME_IN` values for required fields)

## Step 1 — Check Prerequisites

```bash
hurl --version
```

If hurl is not installed, tell the user and stop. Provide the install command for their platform:
- macOS: `brew install hurl`
- Linux: `cargo install hurl` or download from https://github.com/Orange-OpenSource/hurl/releases
- Windows: download from https://github.com/Orange-OpenSource/hurl/releases

## Step 2 — Validate vars.env

Read `security_test_cases/vars.env`. Check:
- `base_url` is set and not `FILL_ME_IN`
- `attacker_token` or `username`/`password` are set (depending on what the .hurl files use)
- No required fields contain `FILL_ME_IN`

If any required field is missing, tell the user which ones to fill in before continuing.

## Step 3 — Inventory .hurl Files

List all `.hurl` files in `security_test_cases/`:

```bash
ls security_test_cases/*.hurl
```

Group them:
- **Fast files** — files without known-slow entries (all except `*_slow.hurl`)
- **Slow files** — files named `*_slow.hurl` (cloud metadata, private IP probes that may hang)

## Step 4 — Run Fast Files

Run all non-slow `.hurl` files together:

```bash
hurl --variables-file security_test_cases/vars.env \
     --test \
     security_test_cases/api1_bola_idor.hurl \
     security_test_cases/api2_broken_authentication.hurl \
     ... (all non-slow .hurl files)
```

Or use glob (if shell supports it):

```bash
hurl --variables-file security_test_cases/vars.env --test \
     $(ls security_test_cases/*.hurl | grep -v slow)
```

Capture exit code:
- `0` — all assertions passed
- `3` — one or more assertion failures (findings)
- `4` — runtime error (connection refused, invalid syntax)

## Step 5 — Handle Failures

For each failed entry, run that single file in verbose mode to see the full request/response:

```bash
hurl --variables-file security_test_cases/vars.env --verbose \
     security_test_cases/api<N>_<name>.hurl 2>&1
```

Read the assertion error message. Determine whether the failure is:
- **A confirmed vulnerability** — the API responded in a way the test documented as insecure (e.g. 200 instead of 403 on an IDOR probe)
- **An assertion mismatch** — the API responded differently than expected but it is not a vulnerability (e.g. 500 instead of 400 for an unsupported scheme)
- **A setup failure** — token expired, server not running, wrong base_url

For assertion mismatches and setup failures, fix the assertion or vars.env and re-run. Do not mark these as vulnerabilities.

## Step 6 — Run Slow Files (Optional)

If `*_slow.hurl` files exist, tell the user:

> "Slow probe files exist (cloud metadata, private IP ranges). These may hang for 30–60s per entry if the server cannot reach those addresses. Run only if you want to test SSRF against cloud/internal infrastructure:"

```bash
hurl --variables-file security_test_cases/vars.env --max-time 30 --test \
     security_test_cases/*_slow.hurl
```

## Step 7 — Generate HTML Report (Optional)

```bash
mkdir -p reports
hurl --variables-file security_test_cases/vars.env \
     --report-html reports/hurl-security-report \
     $(ls security_test_cases/*.hurl | grep -v slow)
```

Then open `reports/hurl-security-report/index.html`.

## Step 8 — Report Findings

After all runs complete, print a findings table:

```
## OWASP API Security Test Results

| Category | File | Result | Finding |
|----------|------|--------|---------|
| API1 BOLA/IDOR | api1_bola_idor.hurl | FAIL | IDOR confirmed on GET /notes?username= |
| API2 Auth       | api2_broken_authentication.hurl | PASS | — |
| API7 SSRF       | api7_ssrf.hurl | FAIL | Self-SSRF via http://127.0.0.1:3000/ |
| ...             | ...            | ...  | ... |
```

For each FAIL entry, include:
- The specific assertion that failed
- The HTTP method + path that triggered it
- The actual response status and relevant body excerpt
- OWASP risk classification (e.g. API1:2023 BOLA/IDOR)

## Step 9 — Suggest Fixes

For each confirmed vulnerability, add a one-line remediation note:
- **IDOR** — enforce object-level ownership check on the server; never trust client-supplied IDs
- **SSRF** — validate and allowlist URLs; block private IP ranges and cloud metadata addresses
- **Broken Auth** — enforce token expiry, reject alg:none, rate-limit login attempts
- **BFLA** — enforce role checks server-side on every privileged endpoint; never rely on client-supplied role fields
- **Mass Assignment** — use an explicit allowlist of writable fields; never pass raw request body to ORM

## Notes

- A PASS on an IDOR probe means the vulnerability was reproduced — read the assertion comment to interpret correctly.
- To regenerate test files: `openapi-owasp-hurl`.
- Next step: `openapi-finding-reporting`.
