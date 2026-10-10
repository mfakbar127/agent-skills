# Cucumber.js JSON Output Schema

The `--format json:<file>` flag writes an array of feature objects.

## Top-level structure

```json
[
  {
    "id": "api1-bola-idor",
    "name": "API1 - Broken Object Level Authorization (BOLA/IDOR)",
    "description": "...",
    "uri": "features/api1_bola_idor.feature",
    "elements": [ /* scenarios */ ]
  }
]
```

## Each element (scenario)

```json
{
  "id": "api1-bola-idor;attacker-reads-victim-s-note-by-manipulating-the-username-query-parameter--idor-",
  "name": "Attacker reads victim's note by manipulating the username query parameter (IDOR)",
  "description": "",
  "type": "scenario",
  "tags": [{ "name": "@api1" }],
  "steps": [ /* step results */ ]
}
```

## Each step result

```json
{
  "name": "I send a GET request to \"/api/getNote?username=victim\"",
  "result": {
    "status": "passed" | "failed" | "pending" | "skipped" | "undefined",
    "error_message": "AssertionError: expected 403 but got 200",
    "duration": 123456789
  }
}
```

## Identifying findings

A scenario is a **confirmed vulnerability** when:
- `result.status === "failed"` AND the scenario name or error_message contains expected patterns like "VULNERABLE", "should be 403", "should not contain"
- OR a step's `result.error_message` indicates the server returned a more permissive response than expected

A scenario is a **marked finding** (VULNERABLE tag) when:
- Any step name contains `the test result should be marked as "VULNERABLE"`
- These steps always pass (they just record the finding) — so the scenario may show `passed` overall but still represents a vulnerability

## Extracting request/response evidence

Step names encode the HTTP call. Parse them to reconstruct evidence:

| Step pattern | What to extract |
|---|---|
| `I send a GET request to "/path"` | Method: GET, Path: /path |
| `I send a POST request to "/path" with body: """{"key":"val"}"""` | Method: POST, Path: /path, Body: {"key":"val"} |
| `the response status should be 200` (failed) | Expected: 200, got: whatever error_message says |
| `the response JSON field "note" should equal "..."` (failed) | The field leaked data it shouldn't |

Response body is not captured by default in Cucumber JSON — note "see test output" if it wasn't captured, and suggest the user add `this.attach(JSON.stringify(this.response.data))` to world.js for richer evidence.
