---
name: security-code-audit-mr
description: This skill should be used when the user asks to "review MR for security", "check PR for vulnerabilities", "security review merge request", "review pull request for security", "audit merge request", or provides a GitLab/GitHub MR/PR URL.
argument-hint: <mr_or_pr_url> [additional_context]
allowed-tools: Bash(grepai *), Bash(git *), Bash(glab *), Bash(gh *), Read, Grep, Glob
---

# MR/PR Security Review

Security review for GitLab Merge Requests and GitHub Pull Requests.

## Purpose

Analyze merge requests and pull requests for security vulnerabilities before merging. Auto-detects platform from URL, fetches diff, and performs comprehensive security analysis.

## Arguments

**$ARGUMENTS** - MR/PR URL (required) + optional additional context

## Usage

```
/security-code-audit-mr https://github.com/owner/repo/pull/123
/security-code-audit-mr https://gitlab.com/owner/repo/-/merge_requests/456
/security-code-audit-mr https://github.com/owner/repo/pull/123 "focus on auth changes"
```

## Workflow

### 1. Parse URL & Detect Platform

Extract from URL:

| Platform | URL Pattern | Extract |
|----------|-------------|---------|
| **GitHub** | `github.com/{owner}/{repo}/pull/{number}` | owner, repo, PR number |
| **GitLab** | `gitlab.com/{owner}/{repo}/-/merge_requests/{number}` | owner, repo, MR number |
| **GitLab Self-hosted** | `gitlab.*` hostname | Same as GitLab |

```bash
# URL parsing example
# GitHub: https://github.com/owner/repo/pull/123
# GitLab: https://gitlab.com/owner/repo/-/merge_requests/456
```

### 2. Verify CLI Tools

```bash
# Check for required CLI
which gh || echo "gh not installed - brew install gh"
which glab || echo "glab not installed - brew install glab"

# Verify authentication
gh auth status 2>/dev/null || echo "GitHub CLI not authenticated"
glab auth status 2>/dev/null || echo "GitLab CLI not authenticated"
```

### 3. Fetch MR/PR Metadata

**GitHub:**
```bash
gh pr view {number} --repo {owner}/{repo} --json title,author,headRefName,baseRefName,state
```

**GitLab:**
```bash
glab mr view {number} --repo {owner}/{repo}
```

### 4. Fetch Diff

**GitHub:**
```bash
gh pr diff {number} --repo {owner}/{repo}
```

**GitLab:**
```bash
glab mr diff {number} --repo {owner}/{repo}
```

**Fallback (if CLI unavailable):**
```bash
git clone --depth 1 {repo_url} /tmp/review_repo
cd /tmp/review_repo
git fetch origin pull/{number}/head:pr-{number}
git diff main...pr-{number}
```

### 5. Parse Changed Files

From diff output:
- File paths: `--- a/file` and `+++ b/file`
- Added lines: `+` prefix
- Removed lines: `-` prefix
- Line numbers: `@@` hunk headers

### 6. Security Analysis

For each changed file:

1. **Detect language** from file extension
2. **Apply language-specific checks** (see `../security-code-audit-review/references/language-checks.md`)
3. **Use grepai for context**:
   ```bash
   grepai search "function context for changed code" --json --compact
   ```
4. **Analyze security implications**:
   - Added lines: May introduce vulnerabilities
   - Removed lines: May fix vulnerabilities
   - Modified lines: May change security properties

### 7. Focus Areas

| Change Type | Priority Checks |
|-------------|-----------------|
| Authentication/authorization | Auth bypass, privilege escalation |
| Input handling | Injection, XSS, path traversal |
| Cryptographic changes | Weak crypto, hardcoded secrets |
| Database operations | SQL injection, data exposure |
| External API calls | SSRF, header injection |

## Output Schema

```json
{
  "mr_pr": {
    "platform": "github",
    "url": "https://github.com/owner/repo/pull/123",
    "number": 123,
    "title": "Add user authentication",
    "author": "developer",
    "source_branch": "feature/auth",
    "target_branch": "main",
    "state": "open"
  },
  "reviews": [
    {
      "file": "src/api/handlers.py",
      "changes": {
        "added_lines": [45, 46, 47],
        "removed_lines": [42]
      },
      "reviews": [
        {
          "issue": "SQL Injection",
          "code_snippet": "cursor.execute(f\"SELECT * FROM users WHERE id = {user_id}\")",
          "reasoning": "Line 46 uses f-string for SQL query with unsanitized user_id",
          "mitigation": "Use parameterized queries: cursor.execute(\"SELECT * FROM users WHERE id = %s\", (user_id,))",
          "confidence": 0.95,
          "cwe": "CWE-89",
          "severity": "CRITICAL",
          "line_number": 46
        }
      ]
    }
  ],
  "summary": {
    "files_changed": 3,
    "additions": 25,
    "deletions": 5,
    "total_issues": 2,
    "by_severity": {
      "CRITICAL": 1,
      "HIGH": 0,
      "MEDIUM": 1,
      "LOW": 0
    },
    "potential_fixes": [
      "Removed insecure eval() in utils.py line 12"
    ]
  }
}
```

## Reference Files

See `../security-code-audit-review/references/` for:
- `language-checks.md` - Security checks per language
- `cwe-reference.md` - Severity levels and CWE IDs
- `output-schema.md` - JSON format validation

## Error Handling

| Error | Resolution |
|-------|------------|
| CLI not installed | Provide installation instructions |
| Not authenticated | Guide user through `gh auth login` or `glab auth login` |
| MR/PR not found | Verify URL and permissions |
| Empty diff | MR/PR may have no changes or be already merged |

## Checklist

- [ ] URL parsed and platform detected
- [ ] CLI tool available and authenticated
- [ ] MR/PR metadata fetched
- [ ] Diff retrieved successfully
- [ ] Changed files and lines parsed
- [ ] Language detected for each file
- [ ] grepai context gathered
- [ ] Security implications analyzed
- [ ] Output matches schema format
- [ ] concise & minimal summary includes severity breakdown
