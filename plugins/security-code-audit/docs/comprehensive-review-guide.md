# Comprehensive Review Guide

This guide explains how to perform full codebase security audits using the security-code-audit-review-full skill.

## Overview

A comprehensive security review scans the entire codebase for vulnerabilities, providing complete coverage of all code files. This is ideal for:

- Initial security assessments
- Compliance audits (SOC 2, PCI DSS, etc.)
- Pre-release security checks
- Annual security reviews

## When to Use

| Scenario | Recommended Approach |
|----------|---------------------|
| First security review | Comprehensive |
| Compliance audit | Comprehensive |
| Pre-release check | Comprehensive + Diff |
| Daily development | Focused or Diff |
| Quick vulnerability check | Focused |

## Running a Comprehensive Review

### Step 1: Ensure Index is Current

```bash
# Check grepai status
grepai status

# Update if needed
grepai watch --status
```

### Step 2: Trigger Full Review

Ask Claude:
```
Run a comprehensive security review on the entire codebase
```

Or use the skill directly:
```
/security-code-audit-review-full
```

### Step 3: Review Results

The output includes:

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

## Analysis Scope

### Languages Covered

| Language | Extensions |
|----------|------------|
| Python | `.py` |
| JavaScript/TypeScript | `.js`, `.ts`, `.jsx`, `.tsx` |
| Go | `.go` |
| Rust | `.rs` |
| C/C++ | `.c`, `.h`, `.cpp`, `.cc`, `.hpp` |
| Java | `.java` |
| C# | `.cs` |
| PHP | `.php` |
| Solidity | `.sol` |
| Terraform | `.tf` |
| Zig | `.zig` |

### Vulnerability Categories

1. **Injection**
   - SQL Injection (CWE-89)
   - Command Injection (CWE-78)
   - Code Injection (CWE-94)
   - LDAP Injection

2. **Authentication & Authorization**
   - Authentication Bypass (CWE-287)
   - Missing Authentication (CWE-306)
   - Privilege Escalation

3. **Cross-Site Scripting**
   - Stored XSS (CWE-79)
   - Reflected XSS (CWE-80)
   - DOM-based XSS

4. **Data Exposure**
   - Path Traversal (CWE-22)
   - Information Disclosure (CWE-200)
   - Sensitive Data in Logs

5. **Cryptography**
   - Weak Crypto (CWE-327)
   - Weak RNG (CWE-330)
   - Hardcoded Secrets (CWE-798)

6. **Memory Safety** (C/C++, Rust unsafe)
   - Buffer Overflow (CWE-119)
   - Use After Free (CWE-416)
   - Double Free (CWE-415)

## Performance Optimization

### For Large Codebases

1. **Exclude Directories**
   ```bash
   # In .grepai/config
   exclude = ["node_modules", "vendor", "dist", "build"]
   ```

2. **Parallel Processing**
   - The agent uses `context: fork` for parallel file analysis
   - Each file is processed independently

3. **Incremental Reviews**
   - Run comprehensive review once
   - Use diff reviews for subsequent changes
   - Re-run comprehensive monthly or before releases

## Interpreting Results

### Severity Prioritization

1. **CRITICAL** - Fix immediately
   - Remote code execution
   - Authentication bypass
   - SQL injection in login

2. **HIGH** - Fix within days
   - Privilege escalation
   - Stored XSS
   - Command injection

3. **MEDIUM** - Fix within weeks
   - Information disclosure
   - Path traversal (limited)
   - CSRF

4. **LOW** - Fix eventually
   - Best practice violations
   - Missing security headers
   - Verbose error messages

### Confidence Scores

| Score | Action |
|-------|--------|
| 0.9 - 1.0 | High confidence - likely real vulnerability |
| 0.7 - 0.9 | Probable - verify manually |
| 0.5 - 0.7 | Possible - needs manual review |
| < 0.5 | Do not report |

## Export and Reporting

### JSON Export

Results are in JSON format suitable for:
- CI/CD integration
- Security dashboards
- Issue trackers
- Compliance reports

### Markdown Report

Convert to markdown:
```bash
# Use jq to format
cat security-report.json | jq -r '.reviews[] | "## \(.file)\n\(.reviews[].issue)"'
```

## Best Practices

1. **Schedule Regular Reviews**
   - Weekly: Diff reviews on PRs
   - Monthly: Focused review on critical components
   - Quarterly: Comprehensive codebase audit

2. **Track Findings Over Time**
   - Store reports with timestamps
   - Monitor vulnerability trends
   - Measure remediation time

3. **Integrate with CI/CD**
   ```yaml
   # .github/workflows/security.yml
   - name: Security Review
     run: claude --skill security-code-audit-review-diff HEAD~1
   ```

4. **Combine with Other Tools**
   - Static analysis (Semgrep, CodeQL)
   - Dependency scanning (Dependabot, Snyk)
   - Dynamic testing (OWASP ZAP)
