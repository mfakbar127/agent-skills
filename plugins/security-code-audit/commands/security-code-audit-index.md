---
name: security-code-audit-index
description: Initialize grepai semantic index for the codebase. Run this before first-time security reviews to enable semantic search capabilities.
allowed-tools: Bash(grepai *), Bash(which *), Read
disable-model-invocation: true
---

# Initialize Security Index

Initialize grepai semantic index for the codebase.

## Purpose

Set up the semantic search index that powers all security review capabilities. Must be run before using any security review features.

## Steps

### 1. Check grepai Availability

```bash
which grepai
```

If not installed, direct user to run `/security-code-audit-setup` first.

### 2. Check Current Status

```bash
grepai status
```

### 3. Initialize if Needed

```bash
grepai init --yes
```

### 4. Start Indexing Daemon

```bash
grepai watch --background
```

### 5. Get Status of Indexing Daemon

```bash
grepai watch --status
```

### 6. Report Results

Provide summary including:
- Number of files indexed
- Index status
- Any errors encountered

## Output Format

```
Index Status Report
===================
grepai: [version/status]
Files indexed: [count]
Index location: [path]
Watch daemon: [Running/Stopped]
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| grepai not found | Run `/security-code-audit-setup` first |
| init fails | Check permissions and disk space |
| watch fails | Check for conflicting processes |
