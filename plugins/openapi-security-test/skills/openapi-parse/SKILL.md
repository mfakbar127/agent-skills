---
name: openapi-parse
description: Parse an OpenAPI/Swagger spec and write a structured summary to openapi-summary.md. Step 1 in the openapi-security-test pipeline.
---

# OpenAPI Parse

Extract a structured summary from an OpenAPI spec so downstream skills (dependency mapping, payload extraction, OWASP test generation) have a clean foundation to work from.

## Workflow

1. Locate the spec file — check `openapi.json`, `swagger.json`, `swagger.yaml` in the current directory, or use the path the user provided.
2. Read the file with the Read tool.
3. Extract and output the structured summary below.
4. Save the summary to `openapi-summary.md` in the current directory so other skills can read it without re-parsing.

## Output Format

Produce `openapi-summary.md` with these sections:

```markdown
# OpenAPI Summary

## Meta
- Title: <info.title>
- Version: <info.version>
- Base URL: <servers[0].url>
- Auth schemes: <list from components.securitySchemes>

## Endpoints

| Method | Path | Auth | Tags | Summary |
|--------|------|------|------|---------|
| POST | /api/login | none | User | Login |
| GET | /api/profile | bearerAuth | User | Get profile |
...

## Request Bodies
For each endpoint that has a requestBody, list:
- `METHOD /path` — fields: field1 (type), field2 (type) — example: {...}

## Response Shapes
For each endpoint, list notable response fields (200 and error codes).

## Auth Flow
- Which endpoints are public (no security requirement)
- Which require bearerAuth (or other schemes)
- How to obtain a token (which endpoint)
```

## Notes

- Resolve `$ref` by reading the `components` section.
- For large specs (>500 lines), parse paths first, then components.
- Next step: `openapi-api-dependency`.
