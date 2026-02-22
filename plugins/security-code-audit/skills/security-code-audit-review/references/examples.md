# Security Review Examples

Concrete examples of completed security reviews showing the full workflow.

## Example 1: Python SQL Injection

### Input File: `auth.py`

```python
from flask import request
import sqlite3

def get_user(user_id):
    conn = sqlite3.connect('users.db')
    cursor = conn.cursor()
    query = f"SELECT * FROM users WHERE id = {user_id}"
    cursor.execute(query)
    return cursor.fetchone()

def login():
    username = request.form.get('username')
    password = request.form.get('password')
    conn = sqlite3.connect('users.db')
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE username = '" + username + "' AND password = '" + password + "'")
    user = cursor.fetchone()
    if user:
        return {"status": "logged_in", "user_id": user[0]}
    return {"status": "failed"}, 401
```

### grepai Commands Run

```bash
grepai search "SQL database query execute" --json --compact
grepai trace callers "cursor.execute" --json
grepai search "user input request form parameter" --json --compact
```

### Analysis Process

1. **Detected language**: Python (`.py` extension)
2. **grepai search** found `cursor.execute` calls in auth.py
3. **grepai trace** showed `get_user()` and `login()` both call `cursor.execute`
4. **Input analysis**: `user_id` and `request.form.get()` are user-controlled
5. **Sanitization check**: No parameterized queries between source and sink
6. **Confirmed vulnerability**: String interpolation in SQL queries

### JSON Output

```json
{
  "reviews": [
    {
      "file": "auth.py",
      "file_path": "/app/auth.py",
      "language": "python",
      "reviews": [
        {
          "issue": "SQL Injection via f-string interpolation",
          "code_snippet": "query = f\"SELECT * FROM users WHERE id = {user_id}\"\ncursor.execute(query)",
          "reasoning": "The user_id parameter is directly interpolated into the SQL query without sanitization. An attacker can inject malicious SQL by providing a value like '1 OR 1=1'.",
          "mitigation": "Use parameterized queries: cursor.execute('SELECT * FROM users WHERE id = ?', (user_id,))",
          "confidence": 0.95,
          "cwe": "CWE-89",
          "severity": "CRITICAL"
        },
        {
          "issue": "SQL Injection via string concatenation",
          "code_snippet": "cursor.execute(\"SELECT * FROM users WHERE username = '\" + username + \"' AND password = '\" + password + \"'\")",
          "reasoning": "Username and password are concatenated directly into the SQL query. Attackers can use payloads like ' OR '1'='1'-- to bypass authentication.",
          "mitigation": "Use parameterized queries: cursor.execute('SELECT * FROM users WHERE username = ? AND password = ?', (username, password))",
          "confidence": 0.95,
          "cwe": "CWE-89",
          "severity": "CRITICAL"
        }
      ]
    }
  ]
}
```

---

## Example 2: React XSS Vulnerability

### Input File: `components/UserProfile.tsx`

```tsx
import React from 'react';

interface Props {
  bio: string;
}

export const UserProfile: React.FC<Props> = ({ bio }) => {
  const [comments, setComments] = React.useState<string[]>([]);
  const [input, setInput] = React.useState('');

  const addComment = () => {
    setComments([...comments, input]);
    setInput('');
  };

  return (
    <div className="profile">
      <div dangerouslySetInnerHTML={{ __html: bio }} />
      <input value={input} onChange={(e) => setInput(e.target.value)} />
      <button onClick={addComment}>Add Comment</button>
      {comments.map((comment, i) => (
        <div key={i} dangerouslySetInnerHTML={{ __html: comment }} />
      ))}
    </div>
  );
};
```

### grepai Commands Run

```bash
grepai search "dangerouslySetInnerHTML innerHTML XSS" --json --compact
grepai search "React component props state user input" --json --compact
grepai trace graph "dangerouslySetInnerHTML" --depth 2 --json
```

### Analysis Process

1. **Detected language**: TypeScript/React (`.tsx` extension)
2. **grepai search** found two `dangerouslySetInnerHTML` usages
3. **Source analysis**: `bio` from props, `comment` from user input
4. **Sanitization check**: No DOMPurify or similar library found
5. **Framework consideration**: JSX auto-escaping doesn't apply to dangerouslySetInnerHTML

### JSON Output

```json
{
  "reviews": [
    {
      "file": "components/UserProfile.tsx",
      "file_path": "/app/components/UserProfile.tsx",
      "language": "typescript",
      "reviews": [
        {
          "issue": "Stored XSS via dangerouslySetInnerHTML with user bio",
          "code_snippet": "<div dangerouslySetInnerHTML={{ __html: bio }} />",
          "reasoning": "The bio prop is rendered as raw HTML. If bio contains user-submitted content, attackers can inject malicious scripts.",
          "mitigation": "Sanitize HTML: <div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(bio) }} />",
          "confidence": 0.85,
          "cwe": "CWE-79",
          "severity": "HIGH"
        },
        {
          "issue": "Stored XSS in comment section",
          "code_snippet": "<div key={i} dangerouslySetInnerHTML={{ __html: comment }} />",
          "reasoning": "User comments stored in state and rendered as raw HTML. Classic stored XSS - scripts persist and execute for all viewers.",
          "mitigation": "Use DOMPurify or remove dangerouslySetInnerHTML: <div>{comment}</div>",
          "confidence": 0.90,
          "cwe": "CWE-79",
          "severity": "HIGH"
        }
      ]
    }
  ]
}
```

---

## Example 3: Go Path Traversal

### Input File: `handlers.go`

```go
func ProcessFile(c *gin.Context) {
    filename := c.Query("file")
    out, err := exec.Command("cat", filename).Output()
    if err != nil {
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }
    c.Data(200, "text/plain", out)
}
```

### JSON Output

```json
{
  "reviews": [
    {
      "file": "handlers.go",
      "file_path": "/app/handlers.go",
      "language": "go",
      "reviews": [
        {
          "issue": "Path traversal via command argument",
          "code_snippet": "filename := c.Query(\"file\")\nout, err := exec.Command(\"cat\", filename).Output()",
          "reasoning": "The filename comes from user input and is passed to cat. While not shell injection, cat reads any path including ../../../etc/passwd.",
          "mitigation": "Validate filename: use filepath.Base(), whitelist directories, or use os.Open() with path validation.",
          "confidence": 0.80,
          "cwe": "CWE-22",
          "severity": "HIGH"
        }
      ]
    }
  ]
}
```
