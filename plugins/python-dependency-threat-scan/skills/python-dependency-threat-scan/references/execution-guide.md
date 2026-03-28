# Execution Guide

## Setup Phase: Build Search Context

**CRITICAL**: Execute this phase first to build deduplicated search context.

Build file patterns and search directories once:
- Define `DEP_FILES` array (requirements*.txt, pyproject.toml, etc.)
- Define `VENV_DIRS` array (.venv, venv, env, __pypackages__)
- Get Python sys.path: `python3 -c "import sys; print(' '.join(sys.path))"`
- Build `SEARCH_DIRS` array (current dir + user site-packages + system site-packages + sys.path)
- Build `ALL_DEP_FILES` array by finding all dependency files once

**Use these arrays throughout remaining phases** - do not repeat find commands.

---

## Phase 0: Project Discovery

**Use pre-built arrays from Setup Phase:**
- Display `ALL_DEP_FILES` array (already discovered)
- Find virtual environments using `VENV_DIRS` patterns
- Find .pth files across `SEARCH_DIRS`
- Read dependency file contents
- Run `.venv/bin/pip list` if venv exists

Document all discovered projects in inventory.

---

## Phase 1: System-Wide Inspection

Execute package enumeration commands:
- `pip list` - list all installed packages
- `pip freeze` - get pinned versions
- `python3 -m site` - identify site-packages locations

**Use `SEARCH_DIRS` from Setup Phase** to inspect all locations:
- Iterate through `SEARCH_DIRS` array (includes sys.path)
- Run `ls -la` on each directory
- Check for suspicious packages

Inspect caches:
- `pip cache list` - list cached packages
- `pip cache dir` - get cache location
- `uv cache dir` - get UV cache location
- Search `~/.cache/pip/` and `~/.cache/uv/`

Document system-wide, user, and cached packages in inventory tables.

---

## Phase 2: IOC Matching (Targeted Mode)

Only execute if user provides specific IOCs.

Match discovered packages against IOCs:
- EXACT match (name/version/hash) → HIGH confidence
- FUZZY match (typosquatting / similar name) → MEDIUM confidence
- STRING match (domain/code content) → MEDIUM confidence
- FILE match (artifact found) → HIGH confidence

**Use pre-built arrays for efficient searching:**

Search for IOC domains:
```bash
for dir in "${SEARCH_DIRS[@]}"; do
  grep -rIHn "$IOC_DOMAIN" "$dir" 2>/dev/null
done
```

Search dependency files for IOC packages:
```bash
for file in "${ALL_DEP_FILES[@]}"; do
  grep -Hn "$IOC_PACKAGE" "$file" 2>/dev/null
done
```

Search for file artifacts:
```bash
for dir in "${SEARCH_DIRS[@]}"; do
  find "$dir" -name "$IOC_FILE" 2>/dev/null
done
```

Document all matches with evidence: file path, line number, content snippet, confidence level.

---

## Phase 3: Security Analysis

**Parse dependencies inline** (replaces analyze_dependencies.py):
```bash
for file in "${ALL_DEP_FILES[@]}"; do
  python3 <<EOF
import re
with open("$file") as f:
  for line_num, line in enumerate(f, 1):
    # Parse package name and version
    # Output: Line X: package version [PINNED/UNPINNED]
EOF
done
```

Claude analyzes output for: unpinned dependencies, typosquatting patterns, suspicious package names, duplicates

**Search for dangerous code patterns** (use SEARCH_DIRS):
```bash
PATTERNS=("exec|eval" "base64\.b64decode" "socket\.socket" "subprocess\.Popen")
for pattern in "${PATTERNS[@]}"; do
  for dir in "${SEARCH_DIRS[@]}"; do
    grep -rIHn "$pattern" "$dir" 2>/dev/null | head -20
  done
done
```

**Check persistence mechanisms inline** (replaces check_persistence.sh):
- Systemd: `find ~/.config/systemd/user -name "*.service" -exec grep -l "python" {} \;`
- Startup scripts: `find ~/.config/sysmon ~/.local/bin /tmp/.hidden -name "*.py" 2>/dev/null`
- .pth files: Use SEARCH_DIRS to find and read .pth files
- Cron: `crontab -l | grep -i python`
- Processes: `ps aux | grep -E "[p]ython"`
- Network: `lsof -i -n | grep -i python`

Document findings: type, location, evidence, severity.

---

## Phase 4: Package Investigation Pattern

When investigating suspicious package, **use pre-built arrays for efficiency:**

**Grep** - Search using deduplicated paths:
```bash
PACKAGE="<package_name>"

# Search dependency files (pre-built list)
for file in "${ALL_DEP_FILES[@]}"; do
  grep -Hn "$PACKAGE" "$file" 2>/dev/null
done

# Search all directories (pre-built list)
for dir in "${SEARCH_DIRS[@]}"; do
  grep -rIHn "$PACKAGE" "$dir" 2>/dev/null | head -10
done
```

Capture: file path, line number, context

**Verify** - Check installation status:
```bash
pip show "$PACKAGE"
pip freeze | grep "$PACKAGE"

# Check venv if exists
for venv in $(find . -type d -name ".venv" -o -name "venv" 2>/dev/null); do
  [ -f "$venv/bin/pip" ] && "$venv/bin/pip" show "$PACKAGE"
done
```

Determine: installed (yes/no), version, location (system/user/project)

**Analyze** - Assess scope and impact:
- Scope: system-wide (`/Library/Python`), user (`~/.local/lib`), or project (`.venv`)
- Declaration: declared in manifests, present in lockfiles, or undeclared (shadow dependency)
- Version consistency: mismatches between manifest/lockfile/different scopes
- Impact: number of grep matches, critical code paths, network access capability

Document: file paths, line numbers, command outputs, evidence chain

---

## Risk Assessment Rules

**HIGH Risk:**
- Exact IOC match (malicious package/version/hash identified)
- Malicious domain connections detected in code
- Active persistence mechanisms (systemd services, .pth files, cron jobs)
- Suspicious Python processes running with network connections
- setup.py with dangerous code execution (exec, os.system, subprocess with curl/wget)

**MEDIUM Risk:**
- Fuzzy/typosquatting match to known malicious packages
- Unpinned dependencies with recent malicious versions available
- Direct URL installs without hash verification
- Suspicious code patterns (eval, exec, base64 decoding without context)
- Undeclared installed packages in virtual environments
- Dependency confusion risk (internal package names without index restriction)

**LOW Risk:**
- Missing lockfiles (version drift possible)
- Unpinned dependencies (no known malicious versions)
- Editable installs (`-e` flag)
- Version drift between manifest and lockfile

**Confidence Levels:**
- HIGH: Multiple corroborating evidence sources, exact matches, command execution verified
- MEDIUM: Single evidence source, fuzzy matches, static analysis only
- LOW: Heuristic-based suspicion, contextual patterns without direct evidence

---

## Output Requirements

Always provide:
- Risk level (LOW/MEDIUM/HIGH) and confidence (LOW/MEDIUM/HIGH)
- All findings with: Type, Location, Evidence, Severity, Confidence, Explanation
- Package inventory tables (System, User, Project)
- IOC matches (if applicable): Exact, Fuzzy, Domain, File artifact
- Suspicious patterns with file:line references
- Verification steps: commands executed and results
- Limitations: missing data, scope constraints, failed commands
- Recommendations: manual review steps, additional data needed, remediation

Separate FACT from SUSPICION. Mark findings [STATIC], [SYSTEM], or [CACHE]. Show all command outputs. Never fabricate data.
