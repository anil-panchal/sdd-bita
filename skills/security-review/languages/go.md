# Go Security Patterns

## Framework Detection

| Indicator | Framework |
|-----------|-----------|
| `net/http`, `http.HandleFunc`, `http.ListenAndServe` | Standard library HTTP |
| `github.com/gin-gonic/gin` | Gin |
| `github.com/labstack/echo` | Echo |
| `github.com/gofiber/fiber` | Fiber |
| `google.golang.org/grpc` | gRPC |

---

## Templates and Output Encoding

### Safe Patterns (Do Not Flag)

```go
// SAFE: html/template performs contextual auto-escaping.
tmpl := template.Must(template.ParseFiles("page.html"))
_ = tmpl.Execute(w, map[string]string{"Name": userInput})

// SAFE: Explicit text output.
_, _ = w.Write([]byte(html.EscapeString(userInput)))
fmt.Fprint(w, template.HTMLEscapeString(userInput))
```

### Flag These (Go-Specific)

```go
// XSS: Disables html/template protections.
template.HTML(userInput) // FLAG: Treats input as trusted HTML
template.JS(userInput)   // FLAG
template.URL(userInput)  // FLAG

// XSS: text/template does not provide HTML contextual escaping.
tmpl := texttemplate.Must(texttemplate.ParseFiles("page.html")) // CHECK
_ = tmpl.Execute(w, data)

// XSS: Manual HTML concatenation.
fmt.Fprintf(w, "<div>%s</div>", userInput) // FLAG
```

### Review Notes

- `html/template` is generally safe for HTML responses.
- `text/template` is fine for plain text, config files, or emails, but not for untrusted HTML output.
- `template.HTML`, `template.JS`, and related trusted types should only wrap constant or already-sanitized values.

---

## Database Access

### Safe Patterns (Do Not Flag)

```go
// SAFE: Parameterized query.
rows, err := db.Query("SELECT * FROM users WHERE id = ?", userID)

// SAFE: PostgreSQL-style placeholders.
row := db.QueryRow("SELECT * FROM users WHERE email = $1", email)

// SAFE: Prepared statement with parameters.
stmt, err := db.Prepare("UPDATE users SET name = ? WHERE id = ?")
_, err = stmt.Exec(name, id)
```

### Flag These (Go-Specific)

```go
// SQL injection: String formatting into query text.
query := fmt.Sprintf("SELECT * FROM users WHERE id = %s", id)
rows, err := db.Query(query) // FLAG

// SQL injection: Concatenation.
query := "SELECT * FROM users WHERE name = '" + name + "'"
_, err := db.Query(query) // FLAG

// SQL injection: Unsafe ORDER BY / table names from input.
query := fmt.Sprintf("SELECT * FROM users ORDER BY %s", sortField) // CHECK
query := fmt.Sprintf("SELECT * FROM %s WHERE id = ?", tableName)   // CHECK
```

### Review Notes

- Parameterize values, not SQL structure.
- Placeholders vary by driver (`?`, `$1`, `:name`), but the security rule is the same.
- Dynamic identifiers such as table names, column names, and sort directions require strict allowlists.

---

## Command Execution

### Always Flag

```go
// Command injection.
exec.Command("sh", "-c", userInput).Run()  // FLAG: Critical
exec.Command("bash", "-c", userInput).Run() // FLAG: Critical

cmd := exec.Command("sh", "-c", fmt.Sprintf("grep %s file.txt", userInput))
_ = cmd.Run() // FLAG
```

### Check Context

```go
// CHECK: Safe only if every argument is constant or allowlisted.
exec.Command("grep", userInput, "file.txt") // CHECK

// SAFE: Fixed binary with validated arguments.
allowed := map[string]bool{"json": true, "yaml": true}
if !allowed[format] {
    return errors.New("invalid format")
}
exec.Command("convert", "--format", format, inputFile).Run()
```

---

## Filesystem and Path Traversal

### Flag These

```go
// Path traversal.
data, err := os.ReadFile(userPath) // FLAG if user-controlled

// Path traversal through join.
fullPath := filepath.Join(baseDir, userPath)
http.ServeFile(w, r, fullPath) // CHECK

// Archive extraction without path validation.
target := filepath.Join(destDir, file.Name)
os.WriteFile(target, content, 0o644) // CHECK: Zip slip / tar traversal
```

### Safer Pattern

```go
cleanBase := filepath.Clean(baseDir) + string(os.PathSeparator)
candidate := filepath.Clean(filepath.Join(baseDir, userPath))
if !strings.HasPrefix(candidate, cleanBase) {
    return errors.New("invalid path")
}
data, err := os.ReadFile(candidate)
```

### Review Notes

- `filepath.Clean()` alone is not enough. Ensure the resolved path stays under an expected base directory.
- Uploaded filenames, archive entry names, and request path segments should all be treated as attacker-controlled.

---

## HTTP, SSRF, and Redirects

### Check Context

```go
// SSRF: Investigate user-controlled targets.
resp, err := http.Get(userURL) // CHECK

req, _ := http.NewRequest(http.MethodGet, input.URL, nil)
resp, err := http.DefaultClient.Do(req) // CHECK

// Open redirect.
http.Redirect(w, r, r.URL.Query().Get("next"), http.StatusFound) // FLAG
```

### Safer Pattern

```go
allowedHosts := map[string]bool{
    "api.internal.example": true,
}

u, err := url.Parse(inputURL)
if err != nil || u.Scheme != "https" || !allowedHosts[u.Hostname()] {
    return errors.New("invalid target")
}

client := &http.Client{Timeout: 5 * time.Second}
resp, err := client.Get(u.String())
```

### Review Notes

- Outbound requests are only SSRF if the destination is attacker-controlled or insufficiently constrained.
- Constant URLs and operator-managed configuration are not attacker-controlled by default.

---

## Crypto and Randomness

### Safe Patterns (Do Not Flag)

```go
// SAFE: Cryptographically secure randomness.
token := make([]byte, 32)
_, err := rand.Read(token)

// SAFE: HMAC / secure primitives from crypto packages.
mac := hmac.New(sha256.New, secretKey)
mac.Write(message)
```

### Flag These

```go
// Weak randomness for security-sensitive values.
token := fmt.Sprintf("%d", mathrand.Int63()) // FLAG if used for reset tokens, sessions, API keys

// Weak password hashing.
sum := md5.Sum([]byte(password))  // FLAG if used for password storage
sum = sha1.Sum([]byte(password))  // FLAG if used for password storage

// Hardcoded credentials or keys.
const apiKey = "secret-key" // FLAG
```

### Review Notes

- `crypto/rand` is the right default for secrets, tokens, nonces, and keys.
- `math/rand` may be fine for sampling, shuffling, retries, or tests, but not for security-sensitive values.
- Prefer password hashing algorithms such as bcrypt, scrypt, or Argon2 for stored passwords.

---

## Input Validation and Deserialization

### Safe Patterns (Do Not Flag)

```go
// SAFE: Decode into typed struct and reject unknown fields.
var req CreateUserRequest
dec := json.NewDecoder(r.Body)
dec.DisallowUnknownFields()
if err := dec.Decode(&req); err != nil {
    return err
}
```

### Flag These

```go
// Unsafe YAML usage with user-controlled content.
err := yaml.Unmarshal(userBytes, &cfg) // CHECK: schema confusion / unsafe downstream use

// Generic map decoding without validation.
var body map[string]any
json.NewDecoder(r.Body).Decode(&body) // CHECK

// Reflective dispatch with user-controlled method names.
reflect.ValueOf(service).MethodByName(method).Call(args) // CHECK
```

### Review Notes

- JSON parsing itself is not code execution, but weak validation often enables authorization bypasses and unsafe dispatch.
- Treat decoded maps and `interface{}` values as untrusted until validated.

---

## Review Aids

```bash
# Dependency and vulnerability scanning
govulncheck ./...

# Suspicious constructs
go vet ./...

# Race conditions in exercised code paths
go test -race ./...

# Fuzz targets for parsers and edge-case input handling
go test -fuzz=. ./...
```

---

## rg Patterns

```bash
# Template trust bypasses
rg -n 'template\.(HTML|JS|URL|CSS)\(' -g '*.go'

# SQL string building
rg -n 'fmt\.Sprintf\("SELECT|fmt\.Sprintf\("UPDATE|fmt\.Sprintf\("DELETE|SELECT \* FROM.*\+' -g '*.go'

# Command execution
rg -n 'exec\.Command\("sh", "-c"|exec\.Command\("bash", "-c"' -g '*.go'

# SSRF and redirects
rg -n 'http\.(Get|Post)\(|DefaultClient\.Do\(|http\.Redirect\(' -g '*.go'

# Weak randomness or hashing
rg -n 'math/rand|md5\.Sum|sha1\.Sum' -g '*.go'

# File access
rg -n 'os\.(ReadFile|Open|Create)|filepath\.Join|http\.ServeFile' -g '*.go'
```

---

## References

- [Go Security Best Practices](https://go.dev/doc/security/best-practices)
- [Avoiding SQL Injection Risk in Go](https://go.dev/doc/database/sql-injection)
- [OWASP Go Secure Coding Practices Guide](https://owasp.org/www-project-go-secure-coding-practices-guide)
