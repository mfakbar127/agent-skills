# Language-Specific Security Checks

## C/C++ (*.c, *.h, *.cpp, *.cc, *.hpp)

### Memory Safety
- Buffer overflows (strcpy, sprintf, gets, strcat, vsprintf)
- Use-after-free, double-free
- Off-by-one errors in buffer operations
- Null pointer dereference
- Uninitialized variables

### Integer Issues
- Integer overflow/underflow
- Integer division by zero
- Sign extension problems

### Format String & Race Conditions
- Format string vulnerabilities (printf with user input)
- TOCTOU (time-of-check-time-of-use) race conditions

### Memory Management
- Memory leaks in security contexts
- Dangling pointers

### Dangerous Functions
strcpy(), strcat(), sprintf(), vsprintf(), gets(), scanf(),
memcpy(), memmove(), printf() with user input, strlen() on untrusted data

### CWE Mapping
CWE-119, CWE-120, CWE-125, CWE-787, CWE-416, CWE-415, CWE-190,
CWE-191, CWE-134, CWE-367, CWE-252, CWE-476, CWE-680

## Python (*.py)

### Injection
- SQL injection (f-strings in queries, raw SQL, string concat)
- Command injection (os.system, subprocess shell=True)
- LDAP injection in search filters

### Deserialization & Template Injection
- pickle.loads, pickle.load on untrusted data
- yaml.load (use yaml.safe_load)
- Jinja2 SSTI (render_template_string with user input)
- Mako template injection

### Path & File Operations
- Path traversal (unsanitized file paths)
- XXE (lxml, xml.etree without feature disable)

### Crypto & Timing
- Insecure hashing (MD5/SHA1 for passwords)
- Timing attacks (== vs compare_digest)
- Hardcoded secrets

### Dangerous Functions
os.system(), subprocess.run(shell=True), pickle.loads(),
yaml.load(), eval(), exec(), compile(), render_template_string()

### CWE Mapping
CWE-78, CWE-89, CWE-94, CWE-22, CWE-502, CWE-1336,
CWE-327, CWE-208, CWE-798, CWE-917

## Go (*.go)

### Concurrency
- Race conditions (goroutine data access without sync)
- Goroutine leaks

### Injection
- SQL injection (string concatenation in queries)
- Command injection (exec.Command with user input)
- Template injection (text/template with untrusted input)

### Network & Crypto
- SSRF (http.Get with user-provided URL)
- InsecureSkipVerify: true in TLS config
- Weak crypto (MD5, SHA1, ECB mode)
- Timing attacks (== vs ConstantTimeCompare)

### Pointer Operations
- Unsafe pointer operations (unsafe.Pointer)

### Dangerous Functions
exec.Command(), os/exec, http.Get() with user input,
template.Parse(), unsafe.Pointer, crypto/md5, crypto/sha1

### CWE Mapping
CWE-362, CWE-366, CWE-89, CWE-78, CWE-94, CWE-918,
CWE-327, CWE-208, CWE-754

## Rust (*.rs)

### Unsafe Code
- Unsafe block analysis (deref raw pointers)
- Memory safety issues in unsafe code
- Borrow checker bypasses

### Error Handling
- unwrap() on user input (panic on malicious data)
- expect() without proper error handling
- panic safety in production code

### FFI & Foreign Code
- extern "C" function calls without validation
- FFI boundary issues

### Memory & Resources
- Integer overflow in safe code (review arithmetic)
- Rc<RefCell> reference cycles (memory leaks)
- Iterator with untrusted input sizes

### Dangerous Patterns
unwrap(), expect(), unsafe{}, extern "C", raw pointers *,
std::ptr::null(), transmute()

### CWE Mapping
CWE-119, CWE-120, CWE-125, CWE-476, CWE-190, CWE-401,
CWE-416, CWE-787, CWE-134, CWE-400

## TypeScript/JavaScript (*.ts, *.tsx, *.js, *.jsx)

### XSS & DOM Manipulation
- XSS (innerHTML, dangerouslySetInnerHTML, document.write)
- DOM-based XSS
- DOM clobbering

### Code Injection
- eval(), Function() usage
- Dynamic imports (import(userInput))

### Prototype & Regex
- Prototype pollution
- ReDoS (catastrophic regex backtracking)

### Security Basics
- Insecure random (Math.random for security tokens)
- Hardcoded secrets, API keys
- CSRF considerations (missing token validation)

### Node.js Specific
- child_process with shell=True or user input
- Buffer.allocUnsafe (uninitialized memory)

### Dangerous Functions
eval(), Function(), innerHTML, dangerouslySetInnerHTML,
document.write(), import(), require(userInput),
exec(), spawn(), Math.random()

### CWE Mapping
CWE-79, CWE-94, CWE-1321, CWE-1333, CWE-338, CWE-798,
CWE-352, CWE-90, CWE-400

## PHP (*.php)

### Injection
- SQL injection (mysql_*, string concat queries)
- Command injection (exec, system, shell_exec, passthru)
- Header injection (CRLF in user input)

### File Operations
- File inclusion (include/require with user input)
- Path traversal (unsanitized file paths)
- File upload security (mime type, extension validation)

### XSS & Output
- XSS (echo without escaping, print)
- XXE (simplexml_load_string without disable)

### Type System
- Type juggling (== vs === comparison issues)

### Session & Crypto
- Session security (missing HttpOnly, Secure flags)
- Open redirect (header("Location: " . $userInput))

### Dangerous Functions
eval(), exec(), system(), shell_exec(), passthru(),
include(), require(), include_once(), require_once(),
mysql_query(), unserialize(), extract()

### CWE Mapping
CWE-89, CWE-78, CWE-79, CWE-22, CWE-94, CWE-434,
CWE-113, CWE-601, CWE-502, CWE-480

## Java (*.java)

### Injection
- SQL injection (Statement with string concat, unparameterized queries)
- XML External Entity (XXE) - DocumentBuilderFactory without feature disable
- LDAP injection in search filters

### Code Execution
- Deserialization (ObjectInputStream.readObject, XMLDecoder)
- Reflection abuse (Class.forName with user input)
- ScriptEngine.eval() with untrusted data

### File System
- Path traversal (File with user input, unsanitized paths)

### Configuration
- Hardcoded credentials, API keys
- Weak crypto (MD5, SHA1 for passwords)
- TrustManager accepting all certificates

### Dangerous Functions
Runtime.exec(), ProcessBuilder.start(), Class.forName(),
ObjectInputStream.readObject(), XMLDecoder.readObject(),
ScriptEngine.eval(), URLClassLoader from untrusted sources

### CWE Mapping
CWE-20, CWE-22, CWE-79, CWE-89, CWE-94, CWE-119, CWE-200,
CWE-287, CWE-352, CWE-400, CWE-502, CWE-676, CWE-798, CWE-918

## C# (*.cs, *.cshtml)

### Injection
- SQL injection (string concat in queries, SqlCommand without parameters)
- XSS (Response.Write without encoding, Razor @Html.Raw)
- Command injection (Process.Start with user input)

### Deserialization
- BinaryFormatter.Deserialize (avoid entirely)
- JavaScriptSerializer without validation
- Json.NET TypeNameHandling auto

### File System
- Path traversal (Path.Combine with user input)
- File.ReadAllText with unvalidated paths

### ASP.NET Specific
- Debug mode in production (<compilation debug="true">)
- Missing custom errors
- ViewStateUserKey not set
- Missing AntiForgeryToken

### Dangerous Functions
Process.Start(), File.ReadAllText(), Assembly.Load(),
BinaryFormatter.Deserialize(), Activator.CreateInstance(),
DynamicMethod.CreateDelegate(), SqlCommand with concat

### CWE Mapping
CWE-12, CWE-22, CWE-79, CWE-89, CWE-94, CWE-119, CWE-1174,
CWE-20, CWE-200, CWE-287, CWE-352, CWE-400, CWE-502, CWE-676, CWE-787

## Zig (*.zig)

### Memory Safety
- Buffer overflow (std.mem.copy without length check)
- Use-after-free (accessing freed allocator memory)
- Out-of-bounds slice access

### Pointer Safety
- @ptrFromInt with untrusted addresses
- @ptrCast without alignment validation
- @bitCast for type punning

### Allocator Issues
- Memory exhaustion (alloc without size limit)
- Double-free patterns
- Arena allocator cleanup missing

### Input Validation
- Path traversal in file operations
- Integer overflow in size calculations

### Dangerous Patterns
@ptrFromInt, @ptrCast, @bitCast, raw pointer arithmetic,
alloc without error handling, defer missing after alloc

### CWE Mapping
CWE-119, CWE-120, CWE-125, CWE-20, CWE-22, CWE-787,
CWE-400, CWE-401, CWE-416, CWE-415, CWE-467, CWE-476, CWE-680, CWE-824, CWE-843

## Solidity (*.sol)

### Reentrancy & State
- Reentrancy vulnerabilities
- State changes after external calls
- Unchecked return values

### Arithmetic
- Integer overflow/underflow (pre-0.8.0)

### Access Control
- Access control issues (missing onlyOwner)
- tx.origin usage for authentication

### External Calls
- Unchecked low-level calls (call, delegatecall)
- Gas limitation issues

### Dangerous Patterns
tx.origin, call(), delegatecall(), selfdestruct(),
unchecked blocks (post-0.8.0), block.timestamp manipulation

### CWE Mapping
CWE-841, CWE-685, CWE-284, CWE-670, CWE-694, CWE-1324,
CWE-1259, CWE-820

## Terraform (*.tf)

### IAM & Access
- Overly permissive IAM policies
- Wildcard actions/resources
- Missing resource constraints

### Secrets & Data
- Exposed secrets in state files
- Hardcoded credentials

### Network Exposure
- Public S3 buckets, databases
- Ingress rules with 0.0.0.0/0
- Missing VPC configuration

### Encryption
- Missing encryption at rest
- Missing encryption in transit
- Unencrypted storage

### Configuration
- Insecure network configurations
- Missing logging/monitoring

### Dangerous Patterns
aws_s3_bucket with public access, aws_security_group with 0.0.0.0/0,
aws_iam_policy with *, provider without encryption defaults

### CWE Mapping
CWE-284, CWE-272, CWE-312, CWE-311, CWE-200,
CWE-922, CWE-522, CWE-693
