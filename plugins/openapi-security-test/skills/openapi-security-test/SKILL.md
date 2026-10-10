---
name: openapi-security-test
description: End-to-end orchestrator: OpenAPI spec → OWASP API Top 10 test suite → finding report. Supports Gherkin (Cucumber.js) and Hurl formats. Runs the full pipeline in one command.
---

# OpenAPI Security Test — Full Pipeline Orchestrator

Run all sub-skills in sequence to go from a raw OpenAPI spec to a runnable OWASP API Security Top 10 2023 test suite.

## Test Format Options

Before starting, ask the user which test format they prefer:

- **Gherkin / BDD** (`openapi-owasp-gherkin`) — generates `.feature` files run by Cucumber.js. Best for teams that want readable, business-language test specs.
- **Hurl** (`openapi-owasp-hurl`) — generates `.hurl` files run directly by the `hurl` CLI. Best for teams that want plain-HTTP test scripts with no framework overhead.

If the user does not specify, default to **Gherkin** and mention that Hurl is also available.

## Pipeline Overview

```
Step 1: openapi-parse          → openapi-summary.md
Step 2: openapi-api-dependency → openapi-dependencies.md
Step 3: openapi-test-context   → openapi-test-context.md   (interactive — asks user, extracts payloads, generates stack-aware boundary values)
Step 4a [Gherkin]: openapi-owasp-gherkin  → security_test_cases/api1..api10 .feature files
Step 4b [Hurl]:    openapi-owasp-hurl     → security_test_cases/api1..api10 .hurl files + vars.env
Step 5a [Gherkin only]: openapi-test-runner  → security_test_cases/support/ + cucumber.js + package.json
Step 5b [Hurl only]:    openapi-runner-hurl  → runs .hurl files, reports findings
Step 6  [both]:         openapi-finding-reporting → finding-report.md
```

Steps 1–2 run without user input. Step 3 pauses to collect runtime test data and tech stack info, then auto-generates enriched payloads. Step 4 uses the chosen format. Step 5a scaffolds the Cucumber runner (Gherkin only); Step 5b runs Hurl files directly. Step 6 generates the finding report for both paths.

## Execution

### Step 1 — Parse

Locate and read the OpenAPI spec (`openapi.json`, `swagger.json`, or `swagger.yaml` in the current directory, or user-provided path). Execute the `openapi-parse` skill workflow. Confirm `openapi-summary.md` is written before continuing.

### Step 2 — Dependency Map

Execute the `openapi-api-dependency` skill workflow using `openapi-summary.md`. Confirm `openapi-dependencies.md` is written before continuing.

### Step 3 — Collect Test Context (PAUSE)

**Stop here and execute the `openapi-test-context` skill.**

This skill asks the user for tech stack info (DB type, framework, cloud provider) and runtime data (credentials, tokens, known resource IDs). It also auto-extracts payload fields from the spec and generates stack-aware boundary test values. It writes `openapi-test-context.md`. Wait for the user to provide answers before proceeding.

Tell the user:
> "Steps 1–2 complete. I need a few pieces of information about your tech stack and runtime environment before generating test scenarios. Please answer the following questions..."

Then run `openapi-test-context`.

### Step 4 — Generate OWASP Test Files

Execute the skill for the chosen format. Pass all three context files to each subagent prompt:
- `openapi-summary.md`
- `openapi-dependencies.md`
- `openapi-test-context.md`

**Gherkin (default):** Execute `openapi-owasp-gherkin`. Spawn all 10 subagents in parallel. Confirm all 10 `.feature` files are written to `security_test_cases/`.

**Hurl:** Execute `openapi-owasp-hurl`. Spawn all 10 subagents in parallel. Confirm all 10 `.hurl` files and `security_test_cases/vars.env` are written.

### Step 5 — Run Tests

**Step 5a — Scaffold Test Runner (Gherkin only):** Skip this step if the user chose Hurl.

Execute the `openapi-test-runner` skill workflow. It reads the `security_test_cases/` directory and generates step definitions. Confirm `security_test_cases/support/`, `security_test_cases/support/steps/`, `cucumber.js`, and `package.json` are written.

**Step 5b — Run Hurl Tests (Hurl only):** Skip this step if the user chose Gherkin.

Execute the `openapi-runner-hurl` skill workflow. It runs the `.hurl` files and produces a findings table.

### Step 6 — Generate Finding Report

Execute the `openapi-finding-reporting` skill workflow for both format paths.

- **Gherkin path:** The skill reads `reports/cucumber-report.json`. Confirm that file exists before running — it is written automatically when `npm run test:security` completes.
- **Hurl path:** The skill reads the findings table printed by `openapi-runner-hurl`. Pass it the console output or ask the user to paste it if not already in context.

Confirm `finding-report.md` is written.

## Completion Message

After all steps finish, print the appropriate message for the chosen format.

**Gherkin:**
```
Security test suite ready.

Generated files:
  openapi-summary.md
  openapi-dependencies.md
  openapi-test-context.md
  security_test_cases/api1_bola_idor.feature
  security_test_cases/api2_broken_authentication.feature
  security_test_cases/api3_broken_object_property_level_authorization.feature
  security_test_cases/api4_unrestricted_resource_consumption.feature
  security_test_cases/api5_broken_function_level_authorization.feature
  security_test_cases/api6_unrestricted_business_flows.feature
  security_test_cases/api7_ssrf.feature
  security_test_cases/api8_security_misconfiguration.feature
  security_test_cases/api9_improper_inventory_management.feature
  security_test_cases/api10_unsafe_consumption_of_apis.feature
  security_test_cases/support/world.js
  security_test_cases/support/hooks.js
  security_test_cases/support/steps/
  cucumber.js

To run:
  npm install
  npm run test:security

Then run:
  /openapi-finding-reporting    ← generates finding-report.md
```

**Hurl:**
```
Security test suite ready.

Generated files:
  openapi-summary.md
  openapi-dependencies.md
  openapi-test-context.md
  security_test_cases/vars.env
  security_test_cases/api1_bola_idor.hurl
  security_test_cases/api2_broken_authentication.hurl
  security_test_cases/api3_broken_object_property_level_authorization.hurl
  security_test_cases/api4_unrestricted_resource_consumption.hurl
  security_test_cases/api5_broken_function_level_authorization.hurl
  security_test_cases/api6_unrestricted_business_flows.hurl
  security_test_cases/api7_ssrf.hurl
  security_test_cases/api8_security_misconfiguration.hurl
  security_test_cases/api9_improper_inventory_management.hurl
  security_test_cases/api10_unsafe_consumption_of_apis.hurl

Edit security_test_cases/vars.env to set your base_url and credentials, then run:
  hurl --variables-file security_test_cases/vars.env security_test_cases/*.hurl

Then run:
  /openapi-finding-reporting    ← generates finding-report.md
```

## Error Handling

- If a step fails to produce its output file, re-run that step alone before continuing.
- If `openapi-test-context` is skipped (user provides no answers), proceed with inferred test data from the spec examples.
- If step 5 fails to write any support file, re-run the `openapi-test-runner` skill directly.
