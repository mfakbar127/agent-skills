# Proactive Agent Setup Guide

This guide explains how to configure the security-code-audit-reviewer agent to trigger automatically after code changes.

## Overview

By default, the security-code-audit-reviewer agent is **reactive** - it only runs when explicitly requested. This guide shows how to make it **proactive** so it automatically reviews code after certain events.

## Option 1: PostToolUse Hook

Create a hook that triggers the agent after Write or Edit operations on code files.

### Create the Hook

Create `hooks/hooks.json` in the plugin:

```json
{
  "PostToolUse": [{
    "matcher": "Write|Edit",
    "hooks": [{
      "type": "prompt",
      "prompt": "If the file modified was a code file (*.py, *.go, *.js, *.ts, *.rs, *.java, *.php, *.sol, *.tf), consider running a security review on the changed code using the security-code-audit-reviewer agent. Only suggest this if the changes involve authentication, database access, user input handling, or file operations."
    }]
  }]
}
```

### Hook Behavior

- Triggers after every Write or Edit tool call
- Checks if file extension matches code files
- Suggests security review only for security-sensitive changes
- User can accept or dismiss the suggestion

## Option 2: PreCommit Hook

Run security review before git commits.

### Create the Hook

```json
{
  "PreToolUse": [{
    "matcher": "Bash",
    "hooks": [{
      "type": "prompt",
      "prompt": "If the user is about to run 'git commit', suggest running /security-code-audit-review-diff --staged first to check staged changes for security issues. This is optional - proceed if the user declines."
    }]
  }]
}
```

## Option 3: Periodic Background Agent

For Claude Code with background agent support, configure automatic periodic reviews.

### Agent Configuration

Update the agent's description to include proactive triggering:

```yaml
---
name: security-code-audit-reviewer
description: Use this agent when [existing conditions] OR when:
  - Significant code changes are detected in security-sensitive areas
  - User has made multiple edits to authentication or database code
  - Daily security scan is scheduled

<example>
Context: User just finished editing authentication code
assistant: "I notice you've made several changes to authentication-related files. Would you like me to run a security review?"
<commentary>
Proactive suggestion after detecting security-sensitive code changes.
</commentary>
</example>
---
```

## Configuration Recommendations

### Conservative (Minimal Interruption)

```json
{
  "PostToolUse": [{
    "matcher": "Write|Edit",
    "hooks": [{
      "type": "prompt",
      "prompt": "If changes involve authentication, crypto, or database queries, ask if user wants a security review."
    }]
  }]
}
```

### Moderate (Balanced)

```json
{
  "PostToolUse": [{
    "matcher": "Write|Edit",
    "hooks": [{
      "type": "prompt",
      "prompt": "Check if file is code (*.py, *.go, *.js, *.ts, *.rs, *.java, *.php). If so, scan for: SQL queries, command execution, file operations, user input handling. If found, suggest security review."
    }]
  }]
}
```

### Aggressive (Maximum Coverage)

```json
{
  "PostToolUse": [{
    "matcher": "Write|Edit",
    "hooks": [{
      "type": "prompt",
      "prompt": "After any code file modification, automatically consider if security review is needed. Prioritize: authentication, database, file I/O, external APIs, crypto. Suggest review if any patterns detected."
    }]
  }]
}
```

## Testing Proactive Setup

1. Apply hook configuration
2. Make a code change involving SQL or user input
3. Verify the agent or review is suggested
4. Confirm the suggestion is appropriate for the change

## Disabling Proactive Mode

Remove or comment out the hooks in `hooks/hooks.json`:

```json
{
  "PostToolUse": []
}
```

Or delete the `hooks/` directory entirely.
