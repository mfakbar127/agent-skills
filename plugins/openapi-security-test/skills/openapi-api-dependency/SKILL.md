---
name: openapi-api-dependency
description: Map auth flows and endpoint dependency chains from an OpenAPI spec. Writes openapi-dependencies.md with the correct call sequence and credential requirements for each endpoint. Step 2 in the openapi-security-test pipeline.
---

# OpenAPI API Dependency

Build an ordered dependency graph so tests can be written with correct setup/teardown sequences.

## Workflow

1. **Read input** — check for `openapi-summary.md` first (written by `openapi-parse`). If absent, read the raw spec (`openapi.json` / `swagger.json` / `swagger.yaml`).
2. **Identify public endpoints** — endpoints with no `security` requirement.
3. **Identify auth endpoints** — the endpoint that issues tokens (usually `POST /login` or `POST /auth/token`).
4. **Classify protected endpoints** — group by auth scheme required.
5. **Detect data dependencies** — e.g., a resource ID returned by one endpoint is needed as input to another.
6. **Output** the dependency map to `openapi-dependencies.md`.

## Output Format

Produce `openapi-dependencies.md`:

```markdown
# API Dependency Map

## Auth Flow
1. POST /auth/register — create test account (public)
2. POST /auth/login — obtain token (public, returns: token)
3. All protected endpoints — pass token as `Authorization: Bearer <token>`

## Endpoint Groups

### Public (no auth required)
- POST /auth/register
- POST /auth/login

### Protected (requires: bearerAuth)
- GET /users/me
- POST /items
- ...

## Data Dependencies
- GET /items/{id} requires: id (obtained from POST /items response)
- GET /users/{username} requires: username (path param — use known username)

## Recommended Test Session Setup Sequence
1. POST /auth/register → {"<username_field>": "<testuser>", "<password_field>": "<testpass>"}
2. POST /auth/login → save token from response
3. Use token for all subsequent requests

## Recommended Teardown
- (list any cleanup endpoints, if none: note that test users persist)
```

## Notes

- Mark endpoints that return tokens/IDs used by others as explicit data dependencies.
- Flag state-accumulating endpoints (counters, submissions) as non-idempotent.
- Next step: `openapi-test-context`.
