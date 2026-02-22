# Diff Review Guide

This guide explains how to perform security reviews on git diffs for pre-commit checks and PR reviews.

## Overview

Diff reviews analyze only changed code, making them ideal for:
- Pre-commit hooks
- Pull request reviews
- CI/CD security gates
- Tracking security impact of changes

## When to Use

| Scenario | Command |
|----------|---------|
| Before committing | `/security-code-audit-review-diff --staged` |
| Review last commit | `/security-code-audit-review-diff HEAD~1` |
| Review PR changes | `/security-code-audit-review-diff main` |
| Review multiple commits | `/security-code-audit-review-diff HEAD~5` |

## Running Diff Reviews

### Staged Changes (Pre-Commit)

```bash
/security-code-audit-review-diff
```

Reviews all staged changes before commit.

### Last Commit

```bash
/security-code-audit-review-diff HEAD~1
```

Reviews the most recent commit.

### Against Branch

```bash
/security-code-audit-review-diff main
/security-code-audit-review-diff origin/main
```

Reviews all changes compared to main branch.

### Commit Range

```bash
/security-code-audit-review-diff HEAD~5
/security-code-audit-review-diff v1.0.0..HEAD
```

Reviews changes over a range of commits.

## Analysis Process

### 1. Parse Git Diff

The skill extracts:
- Changed files
- Added lines (+ prefix)
- Removed lines (- prefix)
- Line numbers

### 2. Analyze Changes

**Added Lines** - May introduce vulnerabilities:
```diff
+ query = f"SELECT * FROM users WHERE id = {user_id}"
```

**Removed Lines** - May fix vulnerabilities:
```diff
- eval(user_input)
```

**Modified Lines** - May change security properties:
```diff
- password = hash(password)
+ password = md5(password)  # Weakened!
```

### 3. Security Focus Areas

| Change Type | Security Concern |
|-------------|-----------------|
| New endpoints | Auth bypass, injection |
| Auth changes | Privilege escalation |
| Crypto changes | Weak algorithms |
| DB changes | SQL injection |
| File ops | Path traversal |
| External calls | SSRF, injection |

## Output Format

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
    "potential_fixes": [
      {
        "file": "auth.py",
        "line": 23,
        "description": "Removed hardcoded password - security improvement"
      }
    ]
  }
}
```

## CI/CD Integration

### GitHub Actions

```yaml
name: Security Review

on:
  pull_request:
    branches: [main]

jobs:
  security-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup grepai
        run: |
          curl -sSL https://raw.githubusercontent.com/yoanbernabeu/grepai/main/install.sh | sh
          grepai init --yes

      - name: Security Review
        run: |
          claude --skill security-code-audit-review-diff origin/main > security-report.json

      - name: Upload Report
        uses: actions/upload-artifact@v4
        with:
          name: security-report
          path: security-report.json
```

### Git Pre-Commit Hook

```bash
#!/bin/bash
# .git/hooks/pre-commit

# Stage changes if not already staged
git diff --quiet --cached || git add -A

# Run security review
echo "Running security review on staged changes..."
claude --skill security-code-audit-review-diff --staged

# Exit with appropriate code
exit $?
```

### GitLab CI

```yaml
security-review:
  stage: test
  script:
    - curl -sSL https://raw.githubusercontent.com/yoanbernabeu/grepai/main/install.sh | sh
    - grepai init --yes
    - claude --skill security-code-audit-review-diff $CI_MERGE_REQUEST_DIFF_BASE_SHA
  artifacts:
    paths:
      - security-report.json
  only:
    - merge_requests
```

## Best Practices

### 1. Always Review Before Commit

```bash
# Stage changes
git add -A

# Run security review
/security-code-audit-review-diff

# Commit if clean
git commit -m "Add feature"
```

### 2. Review All PRs

Require security review in branch protection rules:
- All PRs must pass security check
- Critical/High findings block merge
- Medium findings require acknowledgment

### 3. Track Security Fixes

When removing vulnerable code:
```json
{
  "potential_fixes": [
    {
      "file": "auth.py",
      "line": 42,
      "description": "Removed SQL injection vulnerability"
    }
  ]
}
```

### 4. Set Thresholds

Configure quality gates:
```yaml
# Fail on:
# - Any CRITICAL findings
# - More than 2 HIGH findings
# - More than 5 MEDIUM findings
```

## Troubleshooting

### No Changes Detected

```bash
# Ensure changes are staged
git add -A

# Or specify correct diff target
/security-code-audit-review-diff HEAD
```

### Large Diffs

For large PRs, consider:
1. Breaking into smaller PRs
2. Reviewing by file type
3. Using comprehensive review instead

### False Positives

If grepai returns irrelevant results:
1. Check index is current: `grepai watch --status`
2. Update index: `grepai index`
3. Provide more specific context in your request

## Quick Reference

| Command | Purpose |
|---------|---------|
| `/security-code-audit-review-diff` | Staged changes |
| `/security-code-audit-review-diff HEAD~1` | Last commit |
| `/security-code-audit-review-diff main` | vs main branch |
| `/security-code-audit-review-diff HEAD~10` | Last 10 commits |
| `/security-code-audit-review-diff v1.0.0..HEAD` | Since release |
