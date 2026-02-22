---
name: security-code-audit-review-full
description: This skill should be used when the user asks to "full security audit", "complete codebase review", "security scan entire project", "comprehensive security analysis", or needs full coverage of all code files.
allowed-tools: Bash(grepai *), Bash(find *), Read, Grep, Glob
disable-model-invocation: true
context: fork
agent: Explore
---

# Full Codebase Security Review

Perform comprehensive security review of the entire codebase.

## Purpose

Execute complete security audits covering all code files for initial security assessments, compliance reviews, or comprehensive vulnerability discovery.

## Workflow

### 1. Verify grepai Status

```bash
grepai status
```

### 2. Get List of Code Files

```bash
find . -type f \( \
  -name "*.py" -o \
  -name "*.js" -o -name "*.ts" -o -name "*.jsx" -o -name "*.tsx" -o \
  -name "*.go" -o \
  -name "*.rs" -o \
  -name "*.c" -o -name "*.h" -o -name "*.cpp" -o -name "*.cc" -o \
  -name "*.sol" -o \
  -name "*.tf" -o \
  -name "*.php" \
\) -not -path "*/node_modules/*" -not -path "*/.git/*" -not -path "*/vendor/*"
```

### 3. Gather Security Context

```bash
grepai search "authentication and authorization" --json --compact
grepai search "user input handling and validation" --json --compact
grepai search "database queries and SQL operations" --json --compact
grepai search "file system operations" --json --compact
grepai search "external API calls and network requests" --json --compact
grepai search "cryptography and secrets" --json --compact
grepai search "command execution" --json --compact
```

### 4. Analyze Each File

- Detect language from extension
- Apply language-specific checks
- Use grepai trace for sensitive functions
- Document findings

### 5. Aggregate Results

Combine findings into comprehensive report with summary statistics.

## Output Schema

```json
{
  "reviews": [...],
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

## Reference Files

See `../security-code-audit-review/references/` for:
- `language-checks.md` - Security checks per language
- `cwe-reference.md` - CWE IDs and severity levels
- `output-schema.md` - JSON format validation
- `examples.md` - Completed review examples

## Performance Tips

- Use `context: fork` for parallel processing
- Process files by language group
- Prioritize high-risk areas (auth, input handling, DB)
- Skip generated files and dependencies

## Checklist

- [ ] grepai index is current
- [ ] All code files discovered
- [ ] Security context gathered via grepai
- [ ] Each file analyzed with language-specific checks
- [ ] Summary statistics calculated
- [ ] Results aggregated into single report
