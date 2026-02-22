# CWE Reference

## Severity Levels

| Severity | Examples |
|----------|----------|
| **CRITICAL** | Remote code execution, SQL injection, authentication bypass |
| **HIGH** | Privilege escalation, data exfiltration, significant vulnerabilities |
| **MEDIUM** | Information disclosure, minor security misconfigurations |
| **LOW** | Best practice violations, minor issues |

## Severity Assignment

Use this matrix to determine severity:

| Factor | CRITICAL | HIGH | MEDIUM | LOW |
|--------|----------|------|--------|-----|
| **Exploitability** | Trivial, no auth needed | Easy, needs auth | Moderate complexity | Difficult, unlikely |
| **Impact** | RCE, full data breach | Privilege escalation, partial data access | Info disclosure | Best practice issue |
| **Scope** | All users/systems | Multiple users | Single user | Developer only |
| **Attack Vector** | Network, no user interaction | Network, user interaction required | Local access needed | Physical or admin |

### Severity Decision Factors

1. **Ease of exploitation**: Can a novice attacker exploit this?
2. **Authentication required**: Does it require valid credentials?
3. **User interaction**: Must a victim perform an action?
4. **Data sensitivity**: What data can be accessed or modified?
5. **System impact**: Can the attacker gain shell access or pivot?

### Examples

| Vulnerability | Severity | Reasoning |
|---------------|----------|-----------|
| SQL injection in login form | CRITICAL | No auth needed, full database access |
| XSS in admin panel | HIGH | Requires admin session, but affects all users |
| Path traversal reading logs | MEDIUM | Requires auth, limited file access |
| Missing CSRF token on profile update | LOW | Requires victim action, limited impact |

## Common CWE IDs

| Category | CWE IDs |
|----------|---------|
| Injection | CWE-89 (SQL), CWE-78 (OS Command), CWE-94 (Code Injection) |
| Authentication | CWE-287 (Improper Auth), CWE-306 (Missing Auth) |
| XSS | CWE-79 (Stored), CWE-80 (Reflected) |
| Crypto | CWE-327 (Weak Crypto), CWE-330 (Weak RNG) |
| Memory | CWE-119 (Buffer Overflow), CWE-416 (Use After Free) |
| Data | CWE-22 (Path Traversal), CWE-200 (Info Exposure) |
