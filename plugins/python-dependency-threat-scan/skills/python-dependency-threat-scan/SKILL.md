---
name: python-dependency-threat-scan
description: Scan local Python dependencies for supply chain threats and malicious packages through static analysis and system inspection. Use for dependency security audits, malware detection, IOC matching, and compromise investigation of installed packages.
---

# Python Dependency Threat Scanner

Scan local Python dependencies for malicious packages and supply chain threats through static analysis and read-only system inspection. All commands are inline - no external scripts required.

## Objective

Identify compromise indicators through:
- Static dependency inspection and IOC matching
- System-wide Python package and cache inspection (including sys.path)
- Persistence mechanism detection (.pth files, systemd services, cron jobs)

---

## Pre-Analysis Confirmation

Before starting, confirm scope:

**Ask user:** "Do you want to:
1. **General scan** - Comprehensive security audit of all packages
2. **Targeted scan** - Search for specific package(s) or IOCs"

**If General (1):**
- Confirm directory: "Which directory? (Enter for current)"
- Store: mode=General, target_dir=[path]

**If Targeted (2):**
- Ask: "Package(s)? Version(s)? Domains? File artifacts? SHA256 hashes?"
- Store: mode=Targeted, package_names=[], versions=[], domains=[], file_artifacts=[], hashes=[]

---

## Setup Phase: Build Search Context

Before analysis, build reusable file lists and search directories to eliminate duplicate commands.

### 0. Detect pip Command

```bash
# Detect pip or pip3
if command -v pip >/dev/null 2>&1; then
  PIP_CMD="pip"
elif command -v pip3 >/dev/null 2>&1; then
  PIP_CMD="pip3"
else
  PIP_CMD="python3 -m pip"
fi
echo "Using: $PIP_CMD"
```

### 1. Define File Patterns

```bash
# Dependency file patterns
DEP_FILES=(
  "requirements*.txt"
  "pyproject.toml"
  "Pipfile"
  "Pipfile.lock"
  "poetry.lock"
  "uv.lock"
  "setup.py"
  "setup.cfg"
)

# Virtual environment directories
VENV_DIRS=(
  ".venv"
  "venv"
  "env"
  "__pypackages__"
)

# Persistence files
PTH_FILES="*.pth"
```

### 2. Build Search Directories

```bash
# Get Python sys.path
PYTHON_PATHS=$(python3 -c "import sys; print(' '.join(sys.path))" 2>/dev/null)

SEARCH_DIRS_RAW=(
  "."
  "$HOME/.local/lib/python*/site-packages"
  "/Library/Python/*/site-packages"
  "/usr/local/lib/python*/site-packages"
  $PYTHON_PATHS
)

# Expand globs and deduplicate
SEARCH_DIRS=()
for dir_pattern in "${SEARCH_DIRS_RAW[@]}"; do
  for dir in $dir_pattern; do
    [ -d "$dir" ] && SEARCH_DIRS+=("$dir")
  done
done
SEARCH_DIRS=($(printf '%s\n' "${SEARCH_DIRS[@]}" | sort -u))

printf '%s\n' "${SEARCH_DIRS[@]}"
```

### 3. Discover Dependency Files

```bash
ALL_DEP_FILES=()
for pattern in "${DEP_FILES[@]}"; do
  while IFS= read -r file; do
    [ -f "$file" ] && ALL_DEP_FILES+=("$file")
  done < <(find . -type f -name "$pattern" 2>/dev/null)
done

printf '%s\n' "${ALL_DEP_FILES[@]}"
```

**Use `ALL_DEP_FILES` and `SEARCH_DIRS` throughout remaining phases.**

---

## Execution Workflow

### Phase 0: Project Discovery

**Dependency files:**
```bash
printf '%s\n' "${ALL_DEP_FILES[@]}"
```

**Virtual environments:**
```bash
for venv_pattern in "${VENV_DIRS[@]}"; do
  find . -type d -name "$venv_pattern" 2>/dev/null
done
```

**`.pth` files:**
```bash
# Search in standard Python locations
for dir in "${SEARCH_DIRS[@]}"; do
  find "$dir" -maxdepth 3 -name "$PTH_FILES" 2>/dev/null
done

# Search in cache and local directories (common malware locations)
find ~/.cache/uv -name "*.pth" 2>/dev/null
find ~/.local -name "*.pth" 2>/dev/null
```

**Read dependency files:**
```bash
for file in "${ALL_DEP_FILES[@]}"; do
  cat "$file"
done
```

**Read venv packages:**
```bash
for venv_pattern in "${VENV_DIRS[@]}"; do
  for venv in $(find . -type d -name "$venv_pattern" 2>/dev/null); do
    if [ -f "$venv/bin/pip" ]; then
      "$venv/bin/pip" list
    elif [ -f "$venv/bin/pip3" ]; then
      "$venv/bin/pip3" list
    fi
  done
done
```

### Phase 1: System-Wide Inspection

**Installed packages:**
```bash
$PIP_CMD list
$PIP_CMD freeze
python3 -m site
```

**Search directories:**
```bash
for dir in "${SEARCH_DIRS[@]}"; do
  ls -la "$dir" 2>/dev/null | head -20
done
```

**Caches:**
```bash
$PIP_CMD cache list 2>/dev/null | head -20
$PIP_CMD cache dir 2>/dev/null
uv cache dir 2>/dev/null
[ -d ~/.cache/uv ] && ls -la ~/.cache/uv | head -20
```

### Phase 2: IOC Matching (Targeted Mode Only)

Only execute if user provides specific IOCs.

**Search domains:**
```bash
IOC_DOMAIN="<user_provided_domain>"

for dir in "${SEARCH_DIRS[@]}"; do
  grep -rIHn "$IOC_DOMAIN" "$dir" 2>/dev/null | head -50
done

for file in "${ALL_DEP_FILES[@]}"; do
  grep -Hn "$IOC_DOMAIN" "$file" 2>/dev/null
done
```

**Search packages:**
```bash
IOC_PACKAGE="<user_provided_package>"

for file in "${ALL_DEP_FILES[@]}"; do
  grep -Hn "$IOC_PACKAGE" "$file" 2>/dev/null
done

$PIP_CMD show "$IOC_PACKAGE" 2>/dev/null
$PIP_CMD freeze | grep -i "$IOC_PACKAGE"

for dir in "${SEARCH_DIRS[@]}"; do
  grep -rIHn "$IOC_PACKAGE" "$dir" 2>/dev/null | head -20
done
```

**Search versions:**
```bash
IOC_VERSION="<user_provided_version>"

for file in "${ALL_DEP_FILES[@]}"; do
  grep -Hn "$IOC_VERSION" "$file" 2>/dev/null
done
```

**Search file artifacts:**
```bash
IOC_FILE="<user_provided_file_artifact>"

for dir in "${SEARCH_DIRS[@]}"; do
  find "$dir" -name "$IOC_FILE" 2>/dev/null
done
```

**Match confidence:**
- EXACT (name/version/hash match) → HIGH confidence
- FUZZY (typosquatting) → MEDIUM confidence
- STRING (domain in code) → MEDIUM confidence
- FILE (artifact found) → HIGH confidence

### Phase 3: Security Analysis

**Parse dependencies:**
```bash
for file in "${ALL_DEP_FILES[@]}"; do
  python3 <<EOF
import re, sys
try:
  with open("$file") as f:
    for line_num, line in enumerate(f, 1):
      line = line.strip()
      if not line or line.startswith('#') or line.startswith('-'):
        continue
      # Match package name and version spec
      match = re.match(r'^([a-zA-Z0-9][\w\-_.]*(?:\[[^\]]+\])?)(.*)$', line)
      if match:
        pkg_with_extras = match.group(1)
        version_spec = match.group(2).strip()
        pkg = re.sub(r'\[.*\]', '', pkg_with_extras)

        # Determine pinning status
        if '==' in version_spec:
          status = 'PINNED'
        elif any(op in version_spec for op in ['>=', '>', '~=', '*', '']):
          status = 'UNPINNED' if version_spec else 'UNPINNED'
        else:
          status = 'UNKNOWN'

        print(f"  Line {line_num}: {pkg} {version_spec} [{status}]")
except Exception as e:
  print(f"  Error parsing: {e}", file=sys.stderr)
EOF
done
```

Claude will then analyze the output for:
- Typosquatting (comparing against popular packages: requests, numpy, django, flask, boto3, etc.)
- Suspicious names (single letters, random sequences, misleading names)
- Unpinned dependencies (security risk)
- Duplicate packages

**Check persistence:**

*Systemd services:*
```bash
[ -d ~/.config/systemd/user ] && \
find ~/.config/systemd/user -name "*.service" 2>/dev/null | while read svc; do
  grep -qi "python" "$svc" 2>/dev/null && {
    grep -i "ExecStart" "$svc"
    systemctl --user is-active "$(basename $svc)" 2>/dev/null && echo "[ACTIVE]"
  }
done
```

*Startup scripts:*
```bash
for dir in ~/.config/sysmon ~/.local/bin /tmp/.hidden; do
  [ -d "$dir" ] && \
  find "$dir" -name "*.py" 2>/dev/null | while read script; do
    grep -qE "socket|urllib|requests|subprocess|base64" "$script" 2>/dev/null && \
      echo "[SUSPICIOUS] $script"
  done
done
```

*.pth files:*
```bash
# Check standard Python locations
for dir in "${SEARCH_DIRS[@]}"; do
  find "$dir" -maxdepth 2 -name "*.pth" 2>/dev/null | while read pth; do
    cat "$pth" | while read line; do
      echo "$line" | grep -qE "import|exec|eval|os\.|sys\." && \
        echo "[SUSPICIOUS] $pth: $line" || echo "$pth: $line"
    done
  done
done

# Check cache and local directories (common malware locations)
find ~/.cache/uv -name "*.pth" 2>/dev/null | while read pth; do
  cat "$pth" | while read line; do
    echo "$line" | grep -qE "import|exec|eval|os\.|sys\." && \
      echo "[HIGH RISK - UV CACHE] $pth: $line" || echo "[UV CACHE] $pth: $line"
  done
done

find ~/.local -name "*.pth" 2>/dev/null | while read pth; do
  cat "$pth" | while read line; do
    echo "$line" | grep -qE "import|exec|eval|os\.|sys\." && \
      echo "[HIGH RISK - LOCAL] $pth: $line" || echo "[LOCAL] $pth: $line"
  done
done
```

*Cron jobs:*
```bash
CRON_OUTPUT=$(crontab -l 2>/dev/null)
[ -n "$CRON_OUTPUT" ] && \
echo "$CRON_OUTPUT" | grep -i "python" | while read cron; do
  echo "$cron" | grep -qE "curl.*\||wget.*\||base64|exec" && \
    echo "[SUSPICIOUS] $cron" || echo "$cron"
done
```

*Running processes:*
```bash
ps aux | grep -E "[p]ython|[P]ython" | while read proc; do
  echo "$proc" | grep -qE "socket|urllib|base64|/tmp|hidden" && \
    echo "[SUSPICIOUS] $proc"
done

command -v lsof >/dev/null 2>&1 && lsof -i -n 2>/dev/null | grep -i python
```

**Dangerous code patterns:**
```bash
PATTERNS=(
  "exec|eval|compile|__import__"
  "base64\.b64decode"
  "socket\.socket|urllib\.request|requests\.get"
  "subprocess\.Popen|os\.system"
)

for pattern in "${PATTERNS[@]}"; do
  for dir in "${SEARCH_DIRS[@]}"; do
    grep -rIHn "$pattern" "$dir" 2>/dev/null | head -20
  done
done
```

**Check setup.py:**
```bash
find . -name "setup.py" -type f 2>/dev/null | while read setup; do
  grep -nE "exec|eval|os\.system|subprocess|urllib\.request|base64" "$setup" 2>/dev/null
done
```

### Phase 4: Package Investigation

**Search package:**
```bash
PACKAGE="<package_name>"

for file in "${ALL_DEP_FILES[@]}"; do
  grep -Hn "$PACKAGE" "$file" 2>/dev/null
done

for dir in "${SEARCH_DIRS[@]}"; do
  grep -rIHn "$PACKAGE" "$dir" 2>/dev/null | head -10
done
```

**Check installation:**
```bash
$PIP_CMD show "$PACKAGE" 2>/dev/null
$PIP_CMD freeze | grep -i "$PACKAGE"

for venv_pattern in "${VENV_DIRS[@]}"; do
  for venv in $(find . -type d -name "$venv_pattern" 2>/dev/null); do
    if [ -f "$venv/bin/pip" ]; then
      "$venv/bin/pip" show "$PACKAGE" 2>/dev/null
      "$venv/bin/pip" freeze | grep -i "$PACKAGE"
    elif [ -f "$venv/bin/pip3" ]; then
      "$venv/bin/pip3" show "$PACKAGE" 2>/dev/null
      "$venv/bin/pip3" freeze | grep -i "$PACKAGE"
    fi
  done
done
```

**Assess impact:**
- **Scope**: system-wide, user, or project
- **Declaration**: in manifests, lockfiles, or undeclared
- **Version consistency**: mismatches across locations
- **Impact**: code references, critical paths, network capability

---

## Commands

**Allowed:** pip/pip3 list/show/freeze/cache, python3 -m pip, uv pip list/cache, python3 -m site, python3 -c, find, ls, cat, grep, ps aux, lsof, systemctl --user, crontab -l
**Forbidden:** Any commands modifying files, installing/uninstalling packages, changing permissions, writing files

---

## Output Format

```
### SUMMARY
Scan Mode: General | Targeted
Target Directory: [path]
Search Directories: [count] (including sys.path)
IOCs Provided: [if targeted: list]
Risk Level: LOW | MEDIUM | HIGH
Confidence: LOW | MEDIUM | HIGH
Conclusion: [brief summary]

### FINDINGS
For each: Type (Static/System/Cache/Persistence) | Location | Evidence | Severity | Confidence | Explanation

### PACKAGE INVENTORY
System-Wide, User, Project: [tables]

### IOC MATCHES (if applicable)
Exact | Fuzzy | Domain | File artifact matches

### SUSPICIOUS PATTERNS
Grep findings with file:line | Code anomalies

### DEPENDENCY ANALYSIS
Unpinned packages | Typosquatting candidates | Suspicious names | Duplicates

### PERSISTENCE INDICATORS
Systemd | Startup scripts | .pth files | Cron | Processes

### VERIFICATION STEPS
Commands executed: ✓ command - result | ✗ command - failed

### LIMITATIONS
Missing data | Scope constraints | Failed commands

### RECOMMENDATIONS
Manual review | Additional data needed | Remediation (if compromised)
```

---

## References

- `references/execution-guide.md` - Detailed phase execution and risk assessment rules
- `references/common-iocs.md` - IOC patterns (typosquatting, domains, dangerous code, persistence)

---

## Constraints

Separate FACT from SUSPICION. Always show command outputs. Mark findings [STATIC], [SYSTEM], [CACHE], or [PERSISTENCE]. Never fabricate data. Act as forensic analyst: precise, evidence-driven, minimal false positives. If command fails, document and try alternatives.

---

## Notes

- **No external scripts required** - all commands are inline
- **Deduplicated searches** - build file/directory lists once in Setup Phase
- **sys.path integration** - includes Python runtime paths (catches .pth modifications)
- **Portable** - works on any system with bash and Python 3
