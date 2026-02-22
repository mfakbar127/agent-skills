---
name: security-code-audit-review
description: This skill should be used when the user asks to "security review", "vulnerability scan", "security audit", "check for vulnerabilities", "find security issues", "penetration test", "code security", or mentions specific vulnerabilities (SQL injection, XSS, command injection, buffer overflow, reentrancy, CSRF, SSRF, XXE, path traversal, deserialization).
allowed-tools: Bash(grepai *), Read, Grep, Glob
---

# Security Code Review

Security-focused code review system using grepai semantic search and Claude analysis.

## Purpose

Perform comprehensive security reviews to identify vulnerabilities, trace data flows, and provide actionable mitigations across multiple programming languages.

## Prerequisites

```bash
which grepai || echo "grepai not found - run /security-code-audit-setup"
grepai status
```

## Core Workflow

### 1. Initial Context Gathering

Use grepai search to understand:

```bash
grepai search "authentication and authorization" --json --compact
grepai search "user input handling and validation" --json --compact
grepai search "database queries and SQL operations" --json --compact
grepai search "file system operations" --json --compact
grepai search "external API calls" --json --compact
```

### 2. File Analysis Pattern

For each file:

1. Detect language from extension
2. Apply language-specific checks (see `references/language-checks.md`)
3. Use grepai trace for function relationships
4. Identify data flow from user input
5. Document findings in schema format (see `references/output-schema.md`)

### 3. Data Flow Analysis

Use `/security-code-audit-trace` for source-to-sink tracking:

- **Source-to-sink**: Trace user input to dangerous functions
- **Sink-to-source**: Trace backwards from dangerous functions to input

## Reducing False Positives

Before reporting a finding:

1. **Verify data source is user-controlled** - Use grepai trace to confirm
2. **Check for sanitization** - Look for encoding, escaping, validation
3. **Framework protections** - Confirm framework doesn't auto-protect
4. **Set confidence appropriately**:
   - 0.9+ = Confirmed exploit path
   - 0.7-0.9 = Likely vulnerable
   - 0.5-0.7 = Possible issue, needs manual review
   - <0.5 = Do not report

## Triage Priority

1. **Severity**: CRITICAL > HIGH > MEDIUM > LOW
2. **Confidence**: Higher confidence = more likely real
3. **Reachability**: Exposed endpoints vs internal utilities
4. **Fix complexity**: Quick wins vs major refactors

## Output Schema

```json
{
  "reviews": [
    {
      "file": "relative/path/to/file.py",
      "reviews": [
        {
          "issue": "Short description",
          "code_snippet": "exact vulnerable code",
          "reasoning": "why vulnerable and exploitation path",
          "mitigation": "how to fix",
          "confidence": 0.85,
          "cwe": "CWE-89",
          "severity": "HIGH"
        }
      ]
    }
  ]
}
```

## Reference Files

- **`references/language-checks.md`** - Security checks per language
- **`references/cwe-reference.md`** - Severity levels and CWE IDs
- **`references/output-schema.md`** - JSON format with validation
- **`references/examples.md`** - Completed review examples

## Checklist

- [ ] grepai status shows indexed files
- [ ] Language detected from file extension
- [ ] grepai searches run for: auth, input handling, db access, file ops
- [ ] grepai trace run for all dangerous function calls
- [ ] Each finding has: code_snippet, reasoning, mitigation, CWE, severity
- [ ] Confidence score reflects certainty
- [ ] False positive checks performed
- [ ] Output matches schema format
