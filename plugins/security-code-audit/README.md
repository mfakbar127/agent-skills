# Code Security Audit Plugin

Security-focused code review toolkit using grepai semantic search for Claude Code.

## Features

- **Security Code Reviews**: Scan for vulnerabilities (SQL injection, XSS, command injection, etc.)
- **Data Flow Analysis**: Trace user input to dangerous sinks
- **Security Q&A**: Ask questions about security patterns in your codebase
- **Multiple Review Modes**: Single file, full codebase, or git diff

## Prerequisites

- [grepai](https://github.com/yoanbernabeu/grepai) - Semantic code search
- [Ollama](https://ollama.com/) (recommended) - Local embeddings

## Installation

```bash
# Install grepai
brew install yoanbernabeu/tap/grepai  # macOS
# or: curl -sSL https://raw.githubusercontent.com/yoanbernabeu/grepai/main/install.sh | sh

# Install Ollama (recommended)
brew install ollama  # macOS
ollama pull nomic-embed-text
```

## Usage

### Commands

| Command | Description |
|---------|-------------|
| `/security-code-audit-setup` | Setup and verify grepai/Ollama installation |
| `/security-code-audit-index` | Initialize grepai semantic index |
| `/security-code-audit-update` | Update the semantic index |

### Skills (Auto-triggered)

The following skills activate automatically based on context:

| Skill | Triggers |
|-------|----------|
| Security Review | "security review", "vulnerability scan", "check for vulnerabilities" |
| Security Ask | Investigating security patterns, authentication flows |
| Security Trace | Data flow analysis, injection investigation |
| Security Review File | Targeted file security analysis |
| Security Review Diff | Pre-commit reviews, PR security checks |
| Security Review Full | Complete security audits |

### Agent

| Agent | Description |
|-------|-------------|
| `security-code-audit-reviewer` | Autonomous security review agent (reactive) |

## Quick Start

```bash
# 1. Setup
/security-code-audit-setup

# 2. Index your codebase
/security-code-audit-index

# 3. Run a security review
# Just ask: "Run a security review on my authentication code"
```

## Documentation

- [Proactive Agent Setup](docs/proactive-agent-setup.md) - How to enable automatic security reviews
- [Comprehensive Review Guide](docs/comprehensive-review-guide.md) - Full codebase security audits
- [Diff Review Guide](docs/diff-review-guide.md) - Pre-commit and PR security checks

## Supported Languages

Python, JavaScript/TypeScript, Go, Rust, C/C++, Java, C#, PHP, Solidity, Terraform, Zig

## License

MIT
