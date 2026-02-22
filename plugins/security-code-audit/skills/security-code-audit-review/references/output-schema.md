# Output Schema

All security reviews output findings in this format:

## Base Schema

```json
{
  "reviews": [
    {
      "file": "relative/path/to/file.py",
      "file_path": "/absolute/path/to/file.py",
      "reviews": [
        {
          "issue": "Short description of the vulnerability",
          "code_snippet": "exact vulnerable code",
          "reasoning": "why this is vulnerable and how it can be exploited",
          "mitigation": "how to fix the vulnerability",
          "confidence": 0.85,
          "cwe": "CWE-89",
          "severity": "HIGH"
        }
      ]
    }
  ]
}
```

## Review Finding Fields

| Field | Type | Description |
|-------|------|-------------|
| `issue` | string | Short description of the vulnerability |
| `code_snippet` | string | Exact vulnerable code |
| `reasoning` | string | Why this is vulnerable, exploitation path |
| `mitigation` | string | How to fix the vulnerability |
| `confidence` | float | 0.0-1.0 confidence score |
| `cwe` | string | CWE identifier (e.g., "CWE-89") |
| `severity` | string | CRITICAL, HIGH, MEDIUM, or LOW |

## Extended Schemas

### Full Codebase Review

Adds `summary` object:

```json
{
  "summary": {
    "total_files": 50,
    "files_with_issues": 5,
    "total_issues": 12,
    "by_severity": {
      "CRITICAL": 1,
      "HIGH": 3,
      "MEDIUM": 5,
      "LOW": 3
    }
  }
}
```

### Diff Review

Adds `changes` and `line_number`:

```json
{
  "reviews": [
    {
      "changes": {
        "added_lines": [45, 46, 47],
        "removed_lines": [42]
      },
      "reviews": [
        {
          "line_number": 46,
          ...
        }
      ]
    }
  ],
  "diff_summary": {
    "files_changed": 3,
    "additions": 25,
    "deletions": 5,
    "potential_fixes": [...]
  }
}
```

### Single File Review

Adds `language` field:

```json
{
  "file": "auth.py",
  "language": "python",
  "reviews": [...]
}
```

## Validation Rules

### Required Fields

| Field | Format | Validation |
|-------|--------|------------|
| `issue` | string | Non-empty, concise description |
| `code_snippet` | string | Must be exact match from source file |
| `reasoning` | string | Must explain exploitation path |
| `mitigation` | string | Must be actionable, not generic |
| `confidence` | float | Range: 0.0 - 1.0 |
| `severity` | string | One of: `CRITICAL`, `HIGH`, `MEDIUM`, `LOW` |

### Optional Fields

| Field | Format | When to Use |
|-------|--------|-------------|
| `cwe` | string | Format `CWE-XXX` - recommended for all findings |
| `line_number` | integer | Diff reviews and single file reviews |
| `language` | string | Single file reviews |

### Confidence Guidelines

| Score | Meaning |
|-------|---------|
| 0.9 - 1.0 | Confirmed vulnerability, clear exploit path |
| 0.7 - 0.9 | Likely vulnerable, needs verification |
| 0.5 - 0.7 | Possible issue, requires manual review |
| < 0.5 | Speculative, do not report |

### Common Validation Errors

- `code_snippet` not matching source exactly
- Generic `mitigation` like "fix this" or "sanitize input"
- `severity` not matching impact assessment
- Missing `reasoning` for exploitation path
