---
name: security-code-audit-setup
description: Set up and verify the security review system. Checks grepai installation, Ollama for local embeddings, and initializes the security index.
allowed-tools: Bash(which *), Bash(uname *), Bash(curl *), Bash(brew *), Bash(ollama *), Bash(grepai *), Bash(pgrep *), Bash(lsof *), Bash(ls *), Read
---

# Security Review Setup

Complete setup guide for the security review system.

## Purpose

Verify and configure all dependencies needed for security code reviews: grepai for semantic search and Ollama for local embeddings.

## Setup Checklist

Run through each check sequentially. Report status after each step.

### Step 1: Detect Platform

```bash
uname -s
```

Use this to provide platform-specific installation commands:
- **Darwin** → macOS (prefer Homebrew)
- **Linux** → Use shell script
- **Windows** → Use PowerShell (or WSL)

### Step 2: Check grepai Installation

```bash
which grepai && grepai version
```

**If not installed, guide user with platform-specific install:**

| Platform | Command |
|----------|---------|
| macOS | `brew install yoanbernabeu/tap/grepai` |
| Linux/macOS | `curl -sSL https://raw.githubusercontent.com/yoanbernabeu/grepai/main/install.sh \| sh` |
| Windows | `irm https://raw.githubusercontent.com/yoanbernabeu/grepai/main/install.ps1 \| iex` |

### Step 3: Check Ollama (Recommended for Local Embeddings)

```bash
which ollama && curl -s http://localhost:11434/api/tags | head -c 100
```

**If not installed:**

| Platform | Command |
|----------|---------|
| macOS | `brew install ollama` |
| Linux | `curl -fsSL https://ollama.com/install.sh \| sh` |

**After installation:**
```bash
ollama serve &       # Start Ollama daemon
ollama pull nomic-embed-text   # Pull embedding model
```

## Status Report Format

After running all checks, report:

```
## Security Review Setup Status

| Component | Status | Notes |
|-----------|--------|-------|
| grepai | ✅/❌ | version X.Y.Z or "not installed" |
| Ollama | ✅/❌/⚠️ | running/not installed/not running |
| Embedding Model | ✅/❌ | nomic-embed-text available |
| Project Index | ✅/❌ | .grepai/ initialized |

**Next Steps:**
- [any required actions]
```

## Troubleshooting

### grepai not found after install
- Ensure install directory is in PATH: `echo $PATH`
- May need to restart shell or run `source ~/.zshrc` (or `~/.bashrc`)

### Ollama connection refused
- Check if running: `pgrep ollama`
- Start manually: `ollama serve`
- Check port: `lsof -i :11434`

### Embedding model not found
- Pull model: `ollama pull nomic-embed-text`
- List models: `ollama list`

### Index not building
- Check `.grepai/` exists: `ls -la .grepai/`
- Re-initialize: `grepai init`
- Manual index: `grepai index`

## Quick Commands Reference

```bash
# Full setup (macOS with Homebrew)
brew install yoanbernabeu/tap/grepai ollama
ollama serve &
ollama pull nomic-embed-text

# Check everything
grepai version && grepai status
curl -s http://localhost:11434/api/tags
```
