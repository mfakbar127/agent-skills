---
name: security-code-audit-review-diff
description: This skill should be used when the user asks to "review PR for security", "check staged changes", "security review diff", "pre-commit security check", "review recent changes", or analyzing security impact of git diffs.
argument-hint: [diff_target]
allowed-tools: Bash(grepai *), Bash(git diff *), Read, Grep, Glob
---

# Diff/Patch Security Review

Review git diff for security vulnerabilities in changed code.

## Purpose

Analyze git diffs for security implications in pre-commit reviews, PR security checks, or understanding the security impact of recent changes.

## Arguments

**$ARGUMENTS** - Git diff target (default: `--staged`)

## Usage

```
/security-code-audit-review-diff              # staged changes
/security-code-audit-review-diff HEAD~1       # last commit
/security-code-audit-review-diff HEAD~3       # last 3 commits
/security-code-audit-review-diff main         # vs main branch
/security-code-audit-review-diff feature..main  # between branches
```

## Workflow

### 1. Get Git Diff

```bash
git diff $ARGUMENTS
```

### 2. Parse Changed Files

- File paths (`--- a/file`, `+++ b/file`)
- Added lines (`+` prefix)
- Removed lines (`-` prefix)
- Line numbers (`@@` context)

### 3. For Each Changed File

- Get full file content for context
- Use grepai to understand surrounding code
```bash
grepai search "context for changed function" --json --compact
```

### 4. Analyze Security Implications

| Change Type | Security Concern |
|-------------|-----------------|
| **Added lines** | May introduce vulnerabilities |
| **Removed lines** | May fix vulnerabilities |
| **Modified lines** | May change security properties |

### 5. Focus Areas

- Authentication/authorization changes
- Input validation changes
- Cryptographic changes
- Database query changes

## Output Schema

```json
{
  "reviews": [
    {
      "file": "src/api/handlers.py",
      "changes": {
        "added_lines": [45, 46, 47],
        "removed_lines": [42]
      },
      "reviews": [
        {
          "issue": "Command Injection",
          "code_snippet": "os.system(f\"ping {user_input}\")",
          "reasoning": "Line 46 passes unsanitized user input to os.system",
          "mitigation": "Use subprocess with shell=False",
          "confidence": 0.90,
          "cwe": "CWE-78",
          "severity": "CRITICAL",
          "line_number": 46
        }
      ]
    }
  ],
  "diff_summary": {
    "files_changed": 3,
    "additions": 25,
    "deletions": 5,
    "potential_fixes": []
  }
}
```

## Reference Files

See `../security-code-audit-review/references/` for:
- Language-specific checks
- CWE reference
- Output schema validation

## Checklist

- [ ] Git diff obtained successfully
- [ ] Changed files and lines parsed
- [ ] Context gathered via grepai
- [ ] Security implications analyzed
- [ ] Both additions and removals considered
- [ ] Output includes line numbers
