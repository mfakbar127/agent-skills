# agent-skills

Security and productivity plugins for Claude Code.

## Installation

Add the marketplace:

```
/plugin marketplace add mfakbar127/agent-skills
```

Install a plugin:

```
/plugin install security-code-audit@mfakbar127-skills
/plugin install openapi-security-test@mfakbar127-skills
/plugin install python-dependency-threat-scan@mfakbar127-skills
```

## Available Plugins

### security-code-audit

Security-focused code review toolkit using grepai semantic search.

| Skill | Trigger / Usage |
|-------|-----------------|
| `security-code-audit-review` | "security review", "vulnerability scan", "check for vulnerabilities" |
| `security-code-audit-review-file` | Targeted single-file security analysis |
| `security-code-audit-review-diff` | Pre-commit / PR security checks |
| `security-code-audit-review-full` | Complete codebase security audit |
| `security-code-audit-ask` | "check authentication", "how does auth work", "where is input validated" |
| `security-code-audit-trace` | Data flow analysis, injection investigation |
| `security-code-audit-mr` | Merge request security review |

---

### openapi-security-test

End-to-end OpenAPI security testing pipeline covering all OWASP API Top 10 2023 categories.

| Skill | Step | Description |
|-------|------|-------------|
| `openapi-security-test` | — | Full pipeline orchestrator (run this to start) |
| `openapi-parse` | 1 | Parse spec → `openapi-summary.md` |
| `openapi-api-dependency` | 2 | Map auth flows and endpoint dependencies → `openapi-dependencies.md` |
| `openapi-test-context` | 3 | Collect tech stack info, generate stack-aware payloads → `openapi-test-context.md` |
| `openapi-owasp-gherkin` | 4a | Generate Gherkin `.feature` files for all 10 OWASP categories (parallel) |
| `openapi-owasp-hurl` | 4b | Generate Hurl `.hurl` files for all 10 OWASP categories (parallel) |
| `openapi-test-runner` | 5a | Scaffold Cucumber.js step definitions and runner config |
| `openapi-runner-hurl` | 5b | Run `.hurl` files against a live API and print findings table |
| `openapi-finding-reporting` | 6 | Generate structured `finding-report.md` from test output |

---

### python-dependency-threat-scan

Python supply chain security scanner for detecting malicious packages, typosquatting, and persistence mechanisms through static analysis and system inspection.

| Skill | Description |
|-------|-------------|
| `python-dependency-threat-scan` | Scan installed Python packages for supply chain threats, typosquatting, IOCs, and persistence mechanisms |
