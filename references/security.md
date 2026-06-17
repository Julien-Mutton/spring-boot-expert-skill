# Security — Zero-Trust Spring Boot

Deny by default. Authenticate every request. Authorize per *resource*, not just per role. Validate at the boundary. Never trust client-supplied ids.

## Spring Security baseline (Boot 4)

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    @Bean SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(c -> c.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())                 // deny-by-default
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .oauth2ResourceServer(o -> o.jwt(j -> j.jwtAuthenticationConverter(jwtConverter())));
        return http.build();
    }

    @Bean PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); } // or Argon2
}
```

For pure APIs, prefer `STATELESS` sessions + JWT. CSRF protection applies to cookie/session-based, state-changing browser flows.

## Authorization — defeat IDOR

Role checks are necessary, not sufficient. Verify the principal **owns** the referenced object:

```java
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
public User getUser(Long userId) { ... }

// resource-ownership guard for path ids
@PreAuthorize("@labGuard.owns(#labId, authentication)")
```

Scope queries to the user/tenant in the repository layer — `findByIdAndOwnerId(id, principalId)` — so a forged id simply returns nothing.

## JWT validation (full, every request)

Validate **signature against JWKS**, plus `iss`/`aud`/`exp`. Reject anything that fails. Use RS256 (asymmetric) or a ≥256-bit secret from the environment — never a hardcoded short secret. Short-lived access tokens (15-60 min) + refresh tokens.

## OWASP Top 10 — review focus

| Risk | Insecure | Secure |
|------|----------|--------|
| **A01 Broken Access Control** | No checks; URL-only auth | `@PreAuthorize` per resource; ownership-scoped queries |
| **A02 Cryptographic Failures** | Plaintext / MD5 / SHA1 passwords; secrets/PII in logs | BCrypt/Argon2; env-var secrets; TLS enforced; sanitized logs |
| **A03 Injection** | String-concatenated SQL; `Runtime.exec` w/ user input | Parameterized `@Query`/JPA methods; validate before query; use libraries not shell |
| **A04 Insecure Design** | No rate limiting; "user not found" leaks | Rate-limit auth endpoints; account lockout; generic "invalid credentials" |
| **A05 Misconfiguration** | `endpoints.web.exposure.include=*`; H2 console in prod; stack traces to client | Expose only `health,metrics`; debug off in prod; `ProblemDetail` generic messages; security headers (CSP/HSTS/X-Frame-Options); CORS not `*` |
| **A06 Vulnerable Components** | Stale deps | `mvn versions:display-dependency-updates`, dependency-check; BOM-managed versions |
| **A07 Auth Failures** | Weak passwords; predictable session ids | Strong policy; MFA for sensitive ops; `SecureRandom` tokens; 15-30 min timeout |
| **A08 Integrity Failures** | Java deserialization of untrusted input | JSON + `@Valid`; verify dependency checksums |
| **A09 Logging Failures** | No auth logging | Log auth success/failure with user+IP+timestamp; never log secrets/PII |
| **A10 SSRF** | Fetch arbitrary user URL | Whitelist hosts; block localhost/private ranges |

### Parameterized queries (A03)

```java
// ❌ @Query(value = "SELECT * FROM users WHERE username = '" + name + "'", nativeQuery = true)
@Query("SELECT u FROM User u WHERE u.username = :username")
List<User> findByUsername(@Param("username") String username);   // ✅ — or just a derived method
```

### Generic, non-leaking errors (A05)

```java
@ExceptionHandler(Exception.class)
public ProblemDetail handle(Exception ex) {
    log.error("Internal error", ex);                              // detail server-side only
    return ProblemDetail.forStatusAndDetail(
        HttpStatus.INTERNAL_SERVER_ERROR, "An unexpected error occurred");
}
```

## Input validation at the boundary

```java
public record CreateUserRequest(
    @NotBlank @Size(min = 3, max = 50) @Pattern(regexp = "^[a-zA-Z0-9_]+$") String username,
    @NotBlank @Email String email,
    @NotBlank @Size(min = 12) String password) {}

@PostMapping("/users")
public ResponseEntity<User> create(@Valid @RequestBody CreateUserRequest req) { ... }
```

Size-limit every string input; pattern-validate where applicable; write custom `ConstraintValidator`s for complex rules. Validate shape at the edge with Bean Validation; enforce business invariants in domain constructors.

## Checklist

- [ ] `anyRequest().authenticated()`/`denyAll()` — deny by default
- [ ] `@PreAuthorize` on every sensitive endpoint; ownership checked, not just role
- [ ] Queries scoped to user/tenant; client ids never trusted
- [ ] Passwords BCrypt/Argon2; secrets from env, never source/logs
- [ ] JWT signature + `iss`/`aud`/`exp` validated every request
- [ ] All queries parameterized; all input `@Valid` + size-limited
- [ ] Errors generic to client; security headers + TLS on; actuator exposure minimal
- [ ] Auth attempts logged (no sensitive data); deps scanned and current
