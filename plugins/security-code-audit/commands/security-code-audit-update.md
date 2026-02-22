---
name: security-code-audit-update
description: Update the grepai semantic index for the codebase. Run when files have changed and you need fresh semantic search results.
allowed-tools: Bash(grepai *), Bash(pgrep *), Read
disable-model-invocation: true
---

# Update Security Index

Update the grepai semantic index for the codebase.

## Purpose

Refresh the semantic search index when files have been added, modified, or deleted to ensure security reviews have current context.

## Steps

### 1. Check grepai Status

```bash
grepai status
```

### 2. Check if Watch Daemon is Running

```bash
pgrep -f "grepai watch" || echo "Watch not running"
```

### 3. If Watch is Running

Report auto-update status - files are being indexed automatically.

### 4. If Watch is Not Running

```bash
grepai watch &
```

### 5. Report Results

## Output Format

```
Index Update Report
===================
Status: [Updated/Stale/Not Running]
Watch Daemon: [Running/Stopped]
Last Update: [Timestamp]
Files Indexed: [Count]
Changes Since Last Index: [Added/Modified/Deleted]
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Watch not running | Start with `grepai watch &` |
| Index corrupted | Run `grepai init` to reinitialize |
| Slow updates | Check disk I/O, exclude large dirs |
| Missing files | Check .gitignore patterns |
