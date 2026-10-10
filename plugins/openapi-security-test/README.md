# OpenAPI Security Test Plugin

End-to-end security testing pipeline for REST APIs. Takes an OpenAPI/Swagger spec and produces a runnable OWASP API Top 10 2023 test suite in either Gherkin (Cucumber.js) or Hurl format.

## Features

- **Full Pipeline Orchestration**: One command from raw spec to finding report
- **OWASP API Top 10 2023 Coverage**: All 10 categories, 10 parallel subagents
- **Two Test Formats**: Gherkin BDD (`.feature` + Cucumber.js) or plain HTTP (`.hurl` + Hurl CLI)
- **Stack-Aware Payloads**: Boundary values tailored to your DB, framework, and cloud provider
- **Structured Finding Reports**: CVSS scores, evidence, and remediation per confirmed vulnerability

## Prerequisites

**Gherkin path:**
- Node.js + npm

**Hurl path:**
- `hurl` CLI — `brew install hurl` (macOS) or https://hurl.dev/docs/installation.html

## Usage

### Full Pipeline (recommended)

```bash
# Put openapi.json / swagger.yaml in your project directory, then:
# Ask: "Run a security test on my API"
# Or invoke directly:
/openapi-security-test
```

### Individual Skills

| Skill | Step | Description |
|-------|------|-------------|
| `openapi-parse` | 1 | Parse spec → `openapi-summary.md` |
| `openapi-api-dependency` | 2 | Map auth flows and endpoint dependencies → `openapi-dependencies.md` |
| `openapi-test-context` | 3 | Collect tech stack info, generate stack-aware payloads → `openapi-test-context.md` |
| `openapi-owasp-gherkin` | 4a | Generate Gherkin `.feature` files for all 10 OWASP categories (parallel) |
| `openapi-owasp-hurl` | 4b | Generate Hurl `.hurl` files for all 10 OWASP categories (parallel) |
| `openapi-test-runner` | 5a | Scaffold Cucumber.js step definitions and runner config |
| `openapi-runner-hurl` | 5b | Run `.hurl` files against a live API and print findings table |
| `openapi-finding-reporting` | 6 | Generate structured `finding-report.md` from test output |

## Pipeline Overview

```
Step 1: openapi-parse          → openapi-summary.md
Step 2: openapi-api-dependency → openapi-dependencies.md
Step 3: openapi-test-context   → openapi-test-context.md  (interactive)
Step 4: openapi-owasp-gherkin  → security_test_cases/*.feature
     or openapi-owasp-hurl     → security_test_cases/*.hurl + vars.env
Step 5: openapi-test-runner    → Cucumber.js scaffolding
     or openapi-runner-hurl    → live run + findings table
Step 6: openapi-finding-reporting → finding-report.md
```

## License

MIT
