---
name: security-code-audit-trace
description: This skill should be used when the user asks to "trace data flow", "track user input", "find where data goes", "source to sink", "trace from input", "trace to sink", or investigating injection vulnerabilities and sanitization gaps.
argument-hint: <target> [--direction source-to-sink|sink-to-source]
allowed-tools: Bash(grepai *), Read, Grep, Glob
---

# Security Data Flow Trace

Trace data flow for security analysis - source-to-sink and sink-to-source tracking.

## Purpose

Analyze how data flows through the codebase to identify injection vulnerabilities, understand attack surfaces, and find sanitization gaps.

## Arguments

**$ARGUMENTS** - Function name, variable, or data point to trace, optionally with `--direction source-to-sink` (default) or `--direction sink-to-source`

## Usage

```bash
# Trace from user input (source) to dangerous functions (sink)
/security-code-audit-trace "user_input"
/security-code-audit-trace "request.body" --direction source-to-sink

# Trace backwards from dangerous function to user input
/security-code-audit-trace "execute_query" --direction sink-to-source
/security-code-audit-trace "os.system" --direction sink-to-source
/security-code-audit-trace "cursor.execute" --direction sink-to-source
```

## Concepts

| Term | Definition | Example |
|------|------------|---------|
| **Source** | User-controllable input entry point | HTTP params, file reads, env vars |
| **Sink** | Dangerous function that executes/renders data | SQL exec, eval, file write, system |
| **Sanitizer** | Function that cleans/validates data | escape_html, parameterize, validate |

## Common Sources

| Type | Patterns |
|------|----------|
| HTTP Input | `request.args`, `request.body`, `req.params`, `$_GET`, `$_POST` |
| File Input | `read()`, `readline()`, `file.read()`, `fs.readFile()` |
| Environment | `getenv()`, `os.environ`, `process.env` |
| Database | Query results that feed into other queries |
| Deserialization | `pickle.load()`, `yaml.load()`, `JSON.parse()` |

## Common Sinks

| Type | Patterns | Vulnerability |
|------|----------|---------------|
| SQL | `execute()`, `query()`, `executemany()` | SQL Injection |
| Command | `os.system()`, `subprocess`, `exec.Command()` | Command Injection |
| File | `open()`, `write()`, `os.path.join()` | Path Traversal |
| HTML | `innerHTML`, `document.write()`, `echo` | XSS |
| Eval | `eval()`, `exec()`, `Function()` | Code Injection |
| SSRF | `fetch()`, `requests.get()`, `http.Get()` | SSRF |
| Crypto | `encrypt()`, `hash()`, `compare()` | Weak Crypto |

## Workflow

### Source-to-Sink Analysis

1. **Identify source location**
   ```bash
   grepai search "$TARGET definition and assignment" --json --compact
   ```

2. **Trace data flow forward**
   ```bash
   grepai trace callees "$TARGET" --json
   grepai trace graph "$TARGET" --depth 5 --json
   ```

3. **For each function in the call chain**:
   - Check if data is transformed/sanitized
   - Check if data reaches a dangerous sink
   - Identify security boundaries crossed

### Sink-to-Source Analysis

1. **Identify sink location**
   ```bash
   grepai search "$TARGET definition and usage" --json --compact
   ```

2. **Trace data flow backward**
   ```bash
   grepai trace callers "$TARGET" --json
   grepai trace graph "$TARGET" --depth 5 --json
   ```

3. **For each caller in the chain**:
   - Check if caller receives user input
   - Check if data passes through sanitizers
   - Identify all entry points to the sink

## Output Schema

```json
{
  "trace_type": "source-to-sink",
  "target": "user_input",
  "source_location": {
    "file": "src/handlers.py",
    "line": 23,
    "code": "user_input = request.args.get('id')"
  },
  "flows": [
    {
      "sink": "cursor.execute",
      "sink_file": "src/db/queries.py",
      "sink_line": 45,
      "path": [
        {"file": "src/handlers.py", "function": "get_user", "line": 23},
        {"file": "src/services.py", "function": "find_user", "line": 12},
        {"file": "src/db/queries.py", "function": "execute_query", "line": 45}
      ],
      "sanitizers": [],
      "vulnerable": true,
      "severity": "CRITICAL",
      "cwe": "CWE-89"
    }
  ],
  "summary": {
    "total_paths": 3,
    "vulnerable_paths": 1,
    "sanitized_paths": 2
  }
}
```

## Security Checks

When tracing, identify:

- **Missing sanitization** - Data flows source→sink without cleaning
- **Incomplete sanitization** - Sanitizer doesn't cover all attack vectors
- **Type confusion** - Data type changes mid-flow
- **Encoding issues** - Data encoded/decoded multiple times
- **Boundary crossings** - Data crosses trust boundaries

## Checklist

- [ ] Source/sink identified correctly
- [ ] Call graph traced with grepai
- [ ] All paths documented
- [ ] Sanitizers identified (or noted as missing)
- [ ] Severity and CWE assigned
- [ ] Mitigation suggested
