---
name: security-code-audit-ask
description: This skill should be used when the user asks to "check authentication", "understand security patterns", "find input validation", "how does auth work", "where is user input validated", "what encryption is used", or investigating security flows and patterns.
allowed-tools: Bash(grepai *), Read, Grep, Glob
---

# Security Code Q&A

Ask security-focused questions about the codebase using grepai semantic search.

## Purpose

Use semantic search to understand security patterns, investigate authentication flows, find input validation logic, and explore security-related code without knowing exact file locations.

## Usage

```
/security-code-audit-ask How does authentication work?
/security-code-audit-ask Where is user input validated?
/security-code-audit-ask What encryption is used for passwords?
/security-code-audit-ask Are there any SQL queries that might be vulnerable?
```

## Workflow

### 1. Search for Context

```bash
grepai search "<question>" --json --compact
```

### 2. Trace Call Graphs (if needed)

```bash
grepai trace callers "SensitiveFunction" --json
grepai trace graph "TargetFunction" --depth 3 --json
```

### 3. Read Relevant Files

Use Read tool on files identified by grepai results.

### 4. Synthesize Answer

Provide direct answer with file references and security implications.

## Output Format

```
## Answer

[Direct answer to the question]

## Relevant Code

### [File Path]:[Line Numbers]
[Code snippet]

## Security Considerations

- [Potential concern 1]
- [Potential concern 2]
```

## Question Type Detection

| Question Keywords | Search Focus |
|------------------|--------------|
| auth, login, session | Authentication flow, session handling |
| input, validate, sanitize | Input handling, validation functions |
| encrypt, crypto, password | Cryptographic operations, secrets |
| sql, query, database | Database access patterns |
| api, endpoint, route | API handlers, request processing |
| file, path, upload | File operations, path handling |

## Checklist

- [ ] grepai search executed with --json --compact
- [ ] Call graphs traced if function relationships needed
- [ ] Relevant files read for context
- [ ] Security implications identified
- [ ] Answer includes file references
