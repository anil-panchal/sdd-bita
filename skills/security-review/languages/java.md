# Java Security Patterns

## Framework Detection

| Indicator | Framework |
|-----------|-----------|
| `@Controller`, `@RestController`, `SpringApplication`, `@PreAuthorize` | Spring / Spring Boot |
| `HttpServlet`, `Filter`, `web.xml` | Servlet / JSP |
| `EntityManager`, `JpaRepository`, `@Entity` | JPA / Hibernate |
| `JAX-RS`, `@Path`, `@GET`, `@POST` | Jakarta REST |
| `DocumentBuilderFactory`, `SAXParserFactory`, `XMLInputFactory` | XML parsing |

---

## Spring Security and Authorization

### Safe Patterns (Do Not Flag)

```java
// SAFE: Method-level authorization.
@PreAuthorize("hasRole('ADMIN')")
public User getAdminUser(String id) {
    return userService.findById(id);
}

// SAFE: Declarative request authorization.
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated()
);

// SAFE: Bean validation on request bodies.
public ResponseEntity<User> create(@Valid @RequestBody CreateUserRequest request) {
    return ResponseEntity.ok(service.create(request));
}
```

### Flag These

```java
// Authorization bypass: Object loaded without ownership check.
User user = repository.findById(id).orElseThrow(); // CHECK
return user; // CHECK if caller can read other users' records

// Open redirect in Spring MVC.
return "redirect:" + next; // FLAG if next is request-controlled

// Blanket CSRF disable in browser-backed app.
http.csrf(csrf -> csrf.disable()); // CHECK
```

### Review Notes

- Authentication alone is not authorization. Check object ownership and tenant boundaries.
- `@PreAuthorize` and request rules are strong signals, but still verify service-layer checks for IDOR.
- CSRF disablement may be acceptable for non-browser stateless APIs, but is risky for cookie-authenticated web apps.

---

## SQL and Query Construction

### Safe Patterns (Do Not Flag)

```java
// SAFE: PreparedStatement.
PreparedStatement stmt = con.prepareStatement(
    "SELECT * FROM users WHERE id = ?"
);
stmt.setString(1, userId);

// SAFE: Named or positional parameters in JPA.
TypedQuery<User> q = em.createQuery(
    "select u from User u where u.email = :email", User.class
);
q.setParameter("email", email);

// SAFE: Spring Data repository methods.
userRepository.findByEmail(email);
```

### Flag These

```java
// SQL injection.
String sql = "SELECT * FROM users WHERE name = '" + name + "'";
Statement stmt = con.createStatement();
stmt.executeQuery(sql); // FLAG

// JPQL injection.
String q = "select u from User u where u.name = '" + name + "'";
em.createQuery(q).getResultList(); // FLAG

// Dynamic ORDER BY or table names.
String sql = "SELECT * FROM users ORDER BY " + sort; // CHECK
String sql2 = "SELECT * FROM " + table + " WHERE id = ?"; // CHECK
```

### Review Notes

- ORM use reduces risk but does not eliminate it if query text is assembled from input.
- Dynamic identifiers require strict allowlists.

---

## XSS and Template Rendering

### Safe Patterns (Do Not Flag)

```java
// SAFE: Thymeleaf escapes expressions by default.
<span th:text="${user.displayName}"></span>

// SAFE: JSP with output encoding libraries.
<c:out value="${user.displayName}" />
```

### Flag These

```java
// XSS: Unescaped output in JSP/Servlets.
out.println(userInput); // FLAG if writing HTML response
%><%= request.getParameter("q") %><% // FLAG

// Thymeleaf raw HTML rendering.
<div th:utext="${content}"></div> // FLAG unless sanitized

// Direct HTML concatenation.
response.getWriter().write("<div>" + userInput + "</div>"); // FLAG
```

### Review Notes

- Default template escaping is usually safe.
- Raw output helpers (`th:utext`, manual `println`, direct HTML concatenation) should be treated as high risk.

---

## XML and XXE

### Flag These

```java
// XXE: Insecure XML parser defaults.
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
DocumentBuilder builder = dbf.newDocumentBuilder();
Document doc = builder.parse(inputStream); // FLAG

SAXParserFactory spf = SAXParserFactory.newInstance();
SAXParser parser = spf.newSAXParser(); // FLAG if external entities not disabled
```

### Safer Pattern

```java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
dbf.setXIncludeAware(false);
dbf.setExpandEntityReferences(false);

DocumentBuilder builder = dbf.newDocumentBuilder();
Document doc = builder.parse(inputStream);
```

### Review Notes

- XML parsers must explicitly disable external entity resolution unless there is a strong reason not to.
- Review `DocumentBuilderFactory`, `SAXParserFactory`, `XMLInputFactory`, `TransformerFactory`, and XPath usage.

---

## Deserialization

### Always Flag

```java
// Insecure deserialization.
ObjectInputStream ois = new ObjectInputStream(inputStream);
Object obj = ois.readObject(); // FLAG: Critical if input is untrusted
```

### Check Context

```java
// CHECK: JSON binding is not code execution by itself, but validation matters.
UserRequest req = objectMapper.readValue(body, UserRequest.class);

// CHECK: Dangerous if default typing or polymorphic trust is enabled for untrusted input.
objectMapper.activateDefaultTyping(...); // CHECK
```

### Review Notes

- Native Java serialization on untrusted input is high risk and should be treated as critical.
- JSON parsing should still be reviewed for mass assignment, validation gaps, and unsafe polymorphic configuration.

---

## Command Execution and File Access

### Always Flag

```java
// Command injection.
Runtime.getRuntime().exec(userInput); // FLAG: Critical
new ProcessBuilder("sh", "-c", userInput).start(); // FLAG: Critical
```

### Check Context

```java
// CHECK: Safe only if every argument is fixed or allowlisted.
new ProcessBuilder("convert", "--format", format, file).start();

// Path traversal.
Path target = Paths.get(baseDir, userPath);
Files.readString(target); // CHECK
```

### Safer Pattern

```java
Path base = Paths.get(baseDir).normalize().toAbsolutePath();
Path candidate = base.resolve(userPath).normalize();
if (!candidate.startsWith(base)) {
    throw new SecurityException("invalid path");
}
String contents = Files.readString(candidate);
```

---

## Passwords, Secrets, and Crypto

### Safe Patterns (Do Not Flag)

```java
// SAFE: Strong password hashing.
PasswordEncoder encoder = new BCryptPasswordEncoder();
String hash = encoder.encode(password);

// SAFE: Cryptographic randomness.
SecureRandom random = new SecureRandom();
byte[] nonce = new byte[12];
random.nextBytes(nonce);
```

### Flag These

```java
// Weak password hashing.
MessageDigest.getInstance("MD5");  // FLAG if used for passwords
MessageDigest.getInstance("SHA-1"); // FLAG if used for passwords

// Weak randomness for tokens.
Random random = new Random();
String token = Long.toHexString(random.nextLong()); // FLAG

// Hardcoded secret.
String apiKey = "secret-value"; // FLAG
```

### Review Notes

- `SecureRandom` is the right default for tokens, nonces, and keys.
- `Random` may be fine for non-security use cases such as sampling or tests.
- Password hashing should use bcrypt, Argon2, PBKDF2, or another dedicated password KDF.

---

## Review Aids

```bash
# Common Java build and security checks
./gradlew test
./gradlew dependencyCheckAnalyze

mvn test
mvn org.owasp:dependency-check-maven:check
```

---

## rg Patterns

```bash
# SQL and JPQL injection
rg -n 'createStatement\(|executeQuery\(|executeUpdate\(|SELECT \* FROM.*\+|createQuery\(".*"\s*\+' -g '*.java'

# Insecure deserialization
rg -n 'ObjectInputStream|readObject\(' -g '*.java'

# XML parsers
rg -n 'DocumentBuilderFactory|SAXParserFactory|XMLInputFactory|TransformerFactory|XPathFactory' -g '*.java'

# Command execution
rg -n 'Runtime\.getRuntime\(\)\.exec|ProcessBuilder\("sh", "-c"|ProcessBuilder\("bash", "-c"' -g '*.java'

# Weak crypto or randomness
rg -n 'MessageDigest\.getInstance\("MD5"|MessageDigest\.getInstance\("SHA-1"|new Random\(|SecureRandom' -g '*.java'

# Spring authorization
rg -n '@PreAuthorize|authorizeHttpRequests|csrf\(' -g '*.java'

# Raw HTML output
rg -n 'th:utext|out\.println\(|getWriter\(\)\.write\(' -g '*.java' -g '*.jsp' -g '*.html'
```

---

## References

- [OWASP Java Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Java_Security_Cheat_Sheet.html)
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/)
