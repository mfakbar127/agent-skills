---
name: security-code-audit-review-file
description: This skill should be used when the user asks to "review this file for security", "check file for vulnerabilities", "security analyze file", or mentions specific files like "review auth.py", "check login handler".
argument-hint: <file_path>
allowed-tools: Bash(grepai *), Read, Grep, Glob
---

# Single File Security Review

Review a single file for security vulnerabilities.

## Purpose

Perform targeted security analysis of specific files like authentication handlers, database access layers, or API endpoints.

## Arguments

**$ARGUMENTS** - Path to the file to review

## Usage

```
/security-code-audit-review-file src/auth/login.py
/security-code-audit-review-file ./handlers/user_handler.go
```

## Workflow

### 1. Verify File Exists

```bash
test -f "$ARGUMENTS" && echo "File exists" || echo "File not found"
```

### 2. Read File Content

Use Read tool to get the full file content.

### 3. Detect Language

Determine language from file extension:

| Extension | Language |
|-----------|----------|
| `.py` | Python |
| `.go` | Go |
| `.rs` | Rust |
| `.js`, `.ts`, `.jsx`, `.tsx` | JavaScript/TypeScript |
| `.c`, `.h`, `.cpp`, `.cc` | C/C++ |
| `.java` | Java |
| `.cs` | C# |
| `.php` | PHP |
| `.sol` | Solidity |
| `.tf` | Terraform |
| `.zig` | Zig |

### 4. Gather Context via grepai

```bash
grepai search "functions related to this file" --json --compact
grepai trace callers "SensitiveFunction" --json
grepai trace callees "EntryPoint" --json
```

### 5. Apply Security Checks

See `../security-code-audit-review/references/language-checks.md` for language-specific checks.

## Output Schema

```json
{
  "file": "auth.py",
  "file_path": "/absolute/path/to/auth.py",
  "language": "python",
  "reviews": [
    {
      "issue": "SQL Injection",
      "code_snippet": "cursor.execute(f\"SELECT * FROM users WHERE id = {user_id}\")",
      "reasoning": "User-controlled input directly interpolated into SQL query",
      "mitigation": "Use parameterized queries: cursor.execute(\"SELECT * FROM users WHERE id = %s\", (user_id,))",
      "confidence": 0.95,
      "cwe": "CWE-89",
      "severity": "CRITICAL"
    }
  ]
}
```

## Reference Files

See `../security-code-audit-review/references/` for:
- `language-checks.md` - Security checks per language
- `cwe-reference.md` - CWE IDs and severity levels
- `output-schema.md` - JSON format validation

## Checklist

- [ ] File exists and is readable
- [ ] Language detected correctly
- [ ] grepai context gathered
- [ ] Language-specific checks applied
- [ ] Call graphs traced for sensitive functions
- [ ] Output includes language field
