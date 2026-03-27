# Security Review Guide

Security-focused code review checklist based on OWASP Top 10 and best practices.

## Authentication & Authorization

### Authentication
- [ ] Passwords hashed with strong algorithm (bcrypt, argon2)
- [ ] Password complexity requirements enforced
- [ ] Account lockout after failed attempts
- [ ] Secure password reset flow
- [ ] Multi-factor authentication for sensitive operations
- [ ] Session tokens are cryptographically random
- [ ] Session timeout implemented

### Authorization
- [ ] Authorization checks on every request
- [ ] Principle of least privilege applied
- [ ] Role-based access control (RBAC) properly implemented
- [ ] No privilege escalation paths
- [ ] Direct object reference checks (IDOR prevention)
- [ ] API endpoints protected appropriately

### JWT Security
```java
// Bad: Insecure JWT configuration
Jwts.builder()
    .setSubject(userId)
    .signWith(Keys.hmacShaKeyFor("weak-secret".getBytes()))
    .compact();

// Good: Secure JWT configuration
Jwts.builder()
    .setSubject(userId)
    .setIssuedAt(new Date())
    .setExpiration(Date.from(Instant.now().plus(15, ChronoUnit.MINUTES)))
    .setIssuer("your-app")
    .setAudience("your-api")
    .signWith(privateKey, SignatureAlgorithm.RS256)
    .compact();

// Bad: Not verifying JWT properly
Claims claims = Jwts.parser().parseClaimsJwt(token).getBody(); // No signature verification!

// Good: Verify signature and claims
Claims claims = Jwts.parserBuilder()
    .setSigningKey(publicKey)
    .requireIssuer("your-app")
    .requireAudience("your-api")
    .build()
    .parseClaimsJws(token)
    .getBody();
```

## Input Validation

### SQL Injection Prevention
```java
// Bad: Vulnerable to SQL injection
String query = "SELECT * FROM users WHERE id = " + userId;
statement.executeQuery(query);

// Good: Use parameterized queries
PreparedStatement ps = connection.prepareStatement(
    "SELECT * FROM users WHERE id = ?");
ps.setString(1, userId);
ps.executeQuery();

// Good: Use JPA/Hibernate with proper escaping
userRepository.findById(userId);
```

### XSS Prevention
```java
// Bad: Directly outputting user input in HTML
response.getWriter().write("<div>" + userInput + "</div>");

// Good: Escape HTML entities
response.getWriter().write("<div>" + 
    StringEscapeUtils.escapeHtml4(userInput) + "</div>");

// Good: Use template engines with auto-escaping (Thymeleaf)
// th:text automatically escapes
<div th:text="${userInput}"></div>
```

### Command Injection Prevention
```java
// Bad: Vulnerable to command injection
Runtime.getRuntime().exec("convert " + filename + " output.png");

// Good: Use ProcessBuilder with separate arguments
ProcessBuilder pb = new ProcessBuilder("convert", filename, "output.png");
pb.start();

// Good: Validate and sanitize input
if (!filename.matches("[a-zA-Z0-9_.-]+")) {
    throw new IllegalArgumentException("Invalid filename");
}
```

### Path Traversal Prevention
```java
// Bad: Vulnerable to path traversal
String filePath = "./uploads/" + request.getParameter("filename");
File file = new File(filePath);

// Good: Validate and sanitize path
String filename = Paths.get(request.getParameter("filename")).getFileName().toString();
Path filePath = Paths.get("./uploads").resolve(filename).normalize();

// Verify it's still within uploads directory
if (!filePath.startsWith(Paths.get("./uploads").toAbsolutePath())) {
    throw new SecurityException("Invalid path");
}
```

## Data Protection

### Sensitive Data Handling
- [ ] No secrets in source code
- [ ] Secrets stored in environment variables or secret manager
- [ ] Sensitive data encrypted at rest
- [ ] Sensitive data encrypted in transit (HTTPS)
- [ ] PII handled according to regulations (GDPR, etc.)
- [ ] Sensitive data not logged
- [ ] Secure data deletion when required

### Configuration Security
```yaml
# Bad: Secrets in config files
spring:
  datasource:
    password: "super-secret-password"

# Good: Reference environment variables
spring:
  datasource:
    password: ${DATABASE_PASSWORD}
```

### Error Messages
```java
// Bad: Leaking sensitive information
catch (SQLException e) {
    return ResponseEntity.status(500).body(Map.of(
        "error", e.getMessage(),  // Exposes internal details
        "query", sqlQuery         // Exposes database structure
    ));
}

// Good: Generic error messages
catch (SQLException e) {
    log.error("Database error", e);  // Log internally
    return ResponseEntity.status(500).body(Map.of(
        "error", "An unexpected error occurred"
    ));
}
```

## API Security

### Rate Limiting
- [ ] Rate limiting on all public endpoints
- [ ] Stricter limits on authentication endpoints
- [ ] Per-user and per-IP limits
- [ ] Graceful handling when limits exceeded

### CORS Configuration
```java
// Bad: Overly permissive CORS
@CrossOrigin(origins = "*")

// Good: Restrictive CORS
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://your-app.com")
            .allowedMethods("GET", "POST")
            .allowCredentials(true);
    }
}
```

### HTTP Headers
```java
// Security headers configuration in Spring Security
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .headers(headers -> headers
            .contentSecurityPolicy(csp -> csp
                .policyDirectives("default-src 'self'; script-src 'self'"))
            .httpStrictTransportSecurity(hsts -> hsts
                .includeSubDomains(true)
                .maxAgeInSeconds(31536000))
            .frameOptions(frame -> frame.deny())
            .xssProtection(xss -> xss.enable())
            .contentTypeOptions(Customizer.withDefaults())
        )
        .build();
}
```

## Cryptography

### Secure Practices
- [ ] Using well-established algorithms (AES-256, RSA-2048+)
- [ ] Not implementing custom cryptography
- [ ] Using cryptographically secure random number generation
- [ ] Proper key management and rotation
- [ ] Secure key storage (HSM, KMS)

### Common Mistakes
```java
// Bad: Weak random generation
String token = String.valueOf(Math.random());

// Good: Cryptographically secure random
SecureRandom secureRandom = new SecureRandom();
byte[] tokenBytes = new byte[32];
secureRandom.nextBytes(tokenBytes);
String token = Base64.getUrlEncoder().encodeToString(tokenBytes);

// Bad: MD5/SHA1 for passwords
String hash = DigestUtils.md5Hex(password);

// Good: Use BCrypt or Argon2
String hash = new BCryptPasswordEncoder().encode(password);
```

## Dependency Security

### Checklist
- [ ] Dependencies from trusted sources only
- [ ] No known vulnerabilities (mvn dependency-check, OWASP)
- [ ] Dependencies kept up to date
- [ ] Lock files committed (pom.xml with versions, gradle.lockfile)
- [ ] Minimal dependency usage
- [ ] License compliance verified

### Audit Commands
```bash
# Maven
mvn org.owasp:dependency-check-maven:check

# Gradle
gradle dependencyCheckAnalyze

# General
snyk test
```

## Logging & Monitoring

### Secure Logging
- [ ] No sensitive data in logs (passwords, tokens, PII)
- [ ] Logs protected from tampering
- [ ] Appropriate log retention
- [ ] Security events logged (login attempts, permission changes)
- [ ] Log injection prevented

```java
// Bad: Logging sensitive data
log.info("User login: {}, password: {}", email, password);

// Good: Safe logging
log.info("User login attempt", Map.of("email", email, "success", true));
```

## Security Review Severity Levels

| Severity | Description | Action |
|----------|-------------|--------|
| **Critical** | Immediate exploitation possible, data breach risk | Block merge, fix immediately |
| **High** | Significant vulnerability, requires specific conditions | Block merge, fix before release |
| **Medium** | Moderate risk, defense in depth concern | Should fix, can merge with tracking |
| **Low** | Minor issue, best practice violation | Nice to fix, non-blocking |
| **Info** | Suggestion for improvement | Optional enhancement |
