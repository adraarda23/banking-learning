# Phase 8 Mini-Project — Banking Security Hardening (End-to-End)

## Hedef

Phase 7'de yapılan 4-service banking microservice'lerini **production-grade security hardening** ile bitirme. Keycloak IdP + JWT + OAuth2 + encryption + OWASP defenses + audit + SIEM-ready.

## Süre

10-12 gün (günde 3 saat)

## Önbilgi

- Phase 7 mini-project tamam (4 service split)
- Topic 8.2-8.7 bitti

---

## Görev listesi

### 1. Keycloak setup + realm (1.5 gün)

```yaml
# docker-compose.yml
version: '3.8'
services:
  keycloak-postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: ${KEYCLOAK_DB_PASSWORD}
    volumes: ['keycloak-pg:/var/lib/postgresql/data']
  
  keycloak:
    image: quay.io/keycloak/keycloak:24.0
    command: start
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://keycloak-postgres:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: ${KEYCLOAK_DB_PASSWORD}
      KC_HOSTNAME: auth.mavibank.local
      KC_HTTP_ENABLED: "true"          # dev only
      KC_HOSTNAME_STRICT: "false"
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD}
    ports: ['8180:8080']
    depends_on: [keycloak-postgres]

volumes:
  keycloak-pg:
```

**Realm "banking" config:**

- SSL Required: external
- Brute force: ON (5/15 min)
- Token TTL: access 15 min, refresh 60 min
- SSO idle 30 min, max 8 hour
- Email verification: ON
- Login theme: banking-custom

**Clients:**

| Client ID | Type | Auth flow | Scopes |
|---|---|---|---|
| `banking-mobile` | public + PKCE S256 | Authorization Code | openid, profile, account.read, transactions.read, transfer.write, card.read |
| `banking-web` | public + PKCE S256 | Authorization Code | (same) |
| `banking-api` | bearer-only | (resource) | (resource for the above) |
| `teller-app` | confidential | Authorization Code | openid, profile, customer.read, transfer.write, ... |
| `payment-service` | confidential + service account | Client Credentials | internal.account.read, internal.transfer.write |
| `notification-service` | confidential + service account | Client Credentials | internal.notification.send |

**Realm roles:**
- `customer`
- `teller`
- `branch_manager`
- `admin`
- `compliance_officer`

**Composite role:** `teller` includes scopes from `banking-api` (customer.read, transfer.write).

**User attributes mapper:** `tenant`, `branch`, `customer_id`, `mfa_completed` → JWT claim.

**Custom authentication flow:** `Banking-MFA-Browser` — Username/Password → Risk SPI → OTP enforced.

### 2. Spring Security 6 — all 4 services (2 gün)

`account-service`, `transfer-service`, `card-service`, `notification-service` her birinde:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://auth.mavibank.local:8180/realms/banking
          audiences: [banking-api]
```

`SecurityFilterChain` (each service):

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(c -> c.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(a -> a
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt(j -> 
                j.jwtAuthenticationConverter(keycloakJwtConverter())))
            .headers(h -> h
                .frameOptions(f -> f.deny())
                .contentTypeOptions(Customizer.withDefaults())
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true)
                    .maxAgeInSeconds(31536000)
                    .preload(true))
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives("default-src 'none'; frame-ancestors 'none'"))
                .referrerPolicy(r -> r.policy(ReferrerPolicy.NO_REFERRER)));
        return http.build();
    }
    
    @Bean
    public JwtAuthenticationConverter keycloakJwtConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            Set<GrantedAuthority> auth = new HashSet<>();
            Map<String, Object> realm = jwt.getClaim("realm_access");
            if (realm != null) {
                ((List<String>) realm.get("roles")).forEach(r ->
                    auth.add(new SimpleGrantedAuthority("ROLE_" + r)));
            }
            return auth;
        });
        return converter;
    }
}
```

Endpoint security:

```java
@PostMapping("/transfers")
@PreAuthorize("hasRole('customer') and hasAuthority('SCOPE_transfer.write')")
public Transfer transfer(@Valid @RequestBody TransferRequest req,
                        @AuthenticationPrincipal Jwt jwt) {
    UUID userId = UUID.fromString(jwt.getSubject());
    if (req.amount().compareTo(new BigDecimal("10000")) > 0 
        && !jwt.getClaim("mfa_completed").equals(true)) {
        throw new StepUpRequiredException("MFA required for high-value transfer");
    }
    return transferService.transfer(req, userId);
}
```

### 3. Gateway JWT propagation (0.5 gün)

Spring Cloud Gateway gateway/route'lar JWT'i strip değil, propagate. Resource server'lar her biri JWT validate. **No trust** gateway claim.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: account-route
          uri: lb://account-service
          predicates: [Path=/v1/accounts/**]
          filters:
            - name: TokenRelay
            - name: BankingHeaders
```

### 4. Encryption + KMS (1.5 gün)

LocalStack KMS (dev). AWS KMS (production-like).

```yaml
services:
  localstack:
    image: localstack/localstack:latest
    environment:
      SERVICES: kms,s3
      DEFAULT_REGION: eu-central-1
    ports: ['4566:4566']
```

**Setup KEK:**

```bash
aws --endpoint-url=http://localhost:4566 kms create-key \
  --description "Banking PII KEK" \
  --key-usage ENCRYPT_DECRYPT
```

**Customer service:**
- `Customer.tcKimlik`, `email`, `phone` → `@Convert(EncryptedStringConverter)`
- `tcKimlikHash` blind index (HMAC-SHA256)
- `findByTcKimlik(...)` → hash lookup

**Card service:**
- PAN tokenization service
- `card.pan_token` stored, raw PAN sadece tokenization service vault'unda
- Detokenize only via `payment-processor` role + audit

```java
@Service
public class CardTokenizationService {
    
    private final VaultStorage vault;
    
    @PreAuthorize("hasRole('admin') or hasRole('payment_processor')")
    public String detokenize(String token) {
        auditService.log(
            currentUserId(), 
            "DETOKENIZE_PAN", 
            Map.of("token", token));
        return vault.retrieve(token);
    }
}
```

### 5. Logging + masking (0.5 gün)

Logback config + JSON encoder + masking:

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <providers>
                <pattern>
                    <pattern>{"service": "${spring.application.name}"}</pattern>
                </pattern>
                <timestamp/>
                <logLevel/>
                <loggerName/>
                <mdc/>
                <stackTrace/>
                <message/>
            </providers>
        </encoder>
    </appender>
    
    <appender name="MASKING" class="com.bank.logging.MaskingAppender">
        <appender-ref ref="STDOUT"/>
        <patterns>
            <pattern regex="(\\b\\d{11}\\b)" replacement="***-TC-***"/>
            <pattern regex="(\\b\\d{16}\\b)" replacement="****-****-****-PAN"/>
            <pattern regex="(?i)(password=)([^,\\s]+)" replacement="$1***"/>
            <pattern regex="(?i)(authorization:\\s*bearer\\s+)([\\w.-]+)" replacement="$1***"/>
        </patterns>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="MASKING"/>
    </root>
</configuration>
```

### 6. Audit logging + tamper-proof chain (1.5 gün)

`audit-service` (5th microservice, ya da common library).

```sql
CREATE TABLE audit_event (
    id BIGSERIAL PRIMARY KEY,
    occurred_at TIMESTAMPTZ DEFAULT now(),
    service_name VARCHAR(50),
    user_id UUID,
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(50),
    resource_id VARCHAR(100),
    tenant VARCHAR(10),
    ip_address INET,
    user_agent TEXT,
    trace_id VARCHAR(100),
    details JSONB,
    previous_hash CHAR(64),
    current_hash CHAR(64) NOT NULL UNIQUE
);

CREATE INDEX idx_audit_user_time ON audit_event(user_id, occurred_at DESC);
CREATE INDEX idx_audit_action_time ON audit_event(action, occurred_at DESC);
CREATE INDEX idx_audit_trace ON audit_event(trace_id);
```

Kafka topic'e audit event emit (decouple). Audit service consume → hash chain compute → store.

Audit events:
- `LOGIN_SUCCESS`, `LOGIN_FAIL`
- `LOGOUT`
- `PASSWORD_CHANGE`, `MFA_ENROLL`, `MFA_DISABLE`
- `ACCOUNT_VIEW`, `BALANCE_VIEW`
- `TRANSFER_INITIATED`, `TRANSFER_COMPLETED`, `TRANSFER_FAILED`
- `CARD_BLOCK`, `CARD_UNBLOCK`, `PIN_CHANGE`
- `LIMIT_CHANGE`
- `ADMIN_ACTION` (her admin action)
- `PII_DETOKENIZE`, `PII_DECRYPT`
- `HIGH_VALUE_TRANSFER`
- `SUSPICIOUS_ACTIVITY`

Background scheduled job: `verifyChain()` daily → mismatch → SecurityOps alert.

### 7. OWASP defenses — application-wide (1.5 gün)

Phase 7 mini-project'in üstüne:

**A01 IDOR:** Tüm controller'larda `findByIdAndOwnerId` veya `@PreAuthorize` ownership check. Not Found döner.

**A02:** TLS 1.3 (gateway + service mesh internal mTLS).

**A03:** Tüm endpoint'lerde `@Valid`. Custom validator banking IBAN, TC kimlik.

**A04:** Step-up auth (`mfa_completed` claim check yüksek tutar).

**A05:** `application.yml` actuator lockdown:
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
      show-details: never
```

**A06:** Maven OWASP dependency-check:
```xml
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>
    </configuration>
</plugin>
```

CI: `mvn dependency-check:check` her PR.

**A07:** Keycloak handles. Banking MFA flow.

**A08:** No `ObjectInputStream` anywhere. Webhook HMAC for payment provider callback.

**A09:** Audit chain (yukarıda) + Loki/Elasticsearch indexing.

**A10:** Notification service URL preview endpoint allowlist + private IP block.

### 8. Step-up authentication (1 gün)

Banking yüksek değer işlemler ek MFA:

```java
@PostMapping("/transfers")
@PreAuthorize("hasAuthority('SCOPE_transfer.write')")
public Transfer transfer(@Valid @RequestBody TransferRequest req,
                        @AuthenticationPrincipal Jwt jwt) {
    UUID userId = UUID.fromString(jwt.getSubject());
    
    if (requiresStepUp(req)) {
        String stepUpToken = req.stepUpToken();
        if (stepUpToken == null) {
            throw new StepUpRequiredException(
                "Step-up authentication required",
                stepUpChallengeService.create(userId, req));
        }
        stepUpChallengeService.verify(stepUpToken, userId, req);
    }
    
    return transferService.transfer(req, userId);
}

private boolean requiresStepUp(TransferRequest req) {
    return req.amount().compareTo(new BigDecimal("10000")) > 0
        || isInternationalTransfer(req)
        || isToNewBeneficiary(req);
}
```

```java
@Service
public class StepUpChallengeService {
    
    private final RedisTemplate<String, String> redis;
    private final OtpService otpService;
    
    public ChallengeResponse create(UUID userId, TransferRequest req) {
        String challengeId = UUID.randomUUID().toString();
        otpService.sendOtp(userId);
        
        // Store transfer details in Redis (TTL 5 min)
        redis.opsForValue().set(
            "stepup:" + challengeId,
            objectMapper.writeValueAsString(Map.of(
                "userId", userId,
                "transfer", req)),
            Duration.ofMinutes(5));
        
        return new ChallengeResponse(challengeId, "SMS_OTP", "Enter OTP from SMS");
    }
    
    public void verify(String stepUpToken, UUID userId, TransferRequest req) {
        String[] parts = stepUpToken.split(":");   // "challengeId:otp"
        String key = "stepup:" + parts[0];
        String stored = redis.opsForValue().getAndDelete(key);
        
        if (stored == null) throw new InvalidChallengeException();
        
        Map storedData = objectMapper.readValue(stored, Map.class);
        // Verify transfer matches what was challenged
        if (!storedData.get("userId").equals(userId.toString())
            || !storedData.get("transfer").equals(req)) {
            throw new ChallengeMismatchException();
        }
        
        otpService.verifyOtp(userId, parts[1]);
    }
}
```

### 9. Security testing (1.5 gün)

**Integration tests** (25+):
- IDOR reject (different user's account)
- SQL injection reject
- XSS payload escape
- Stack trace not exposed
- Step-up required for high-value
- Refresh token rotation + reuse detection
- Encrypted column DB raw value
- Tokenization roundtrip
- Audit chain verify + tamper detect
- Brute force lockout (Keycloak Testcontainers)
- JWT validation (expired, wrong audience, tampered)
- CORS reject unknown origin
- SSRF reject metadata service
- SSRF reject private IP
- Webhook HMAC verify
- Sensitive header (Authorization) masked in logs
- Actuator /env not accessible without admin
- Method-level @PreAuthorize enforcement
- Composite role permission check
- mTLS service-to-service (if applicable)
- HSTS header present
- CSP header present
- X-Frame-Options DENY
- BCrypt password format
- HaveIBeenPwned k-anonymity check

**DAST:** OWASP ZAP baseline scan against running stack:
```bash
docker run -t owasp/zap2docker-stable zap-baseline.py \
  -t http://gateway:8080 \
  -r zap-report.html
```

**SAST:** SpotBugs FindSecBugs:
```xml
<plugin>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-maven-plugin</artifactId>
    <configuration>
        <plugins>
            <plugin>
                <groupId>com.h3xstream.findsecbugs</groupId>
                <artifactId>findsecbugs-plugin</artifactId>
                <version>1.12.0</version>
            </plugin>
        </plugins>
    </configuration>
</plugin>
```

### 10. Production-grade kasten kırma senaryoları (1 gün)

#### Senaryo 1: IDOR exploit attempt

```bash
TOKEN_USER_A=$(login userA)
# User A account: account-id-A
# User B account: account-id-B

curl -H "Authorization: Bearer $TOKEN_USER_A" \
  https://api.mavibank.local/v1/accounts/account-id-B

→ 404 Not Found (NOT 403, info leak prevention)
Audit log: ACCOUNT_VIEW_ATTEMPT user=A, resource=account-id-B, result=DENIED
```

#### Senaryo 2: Step-up bypass attempt

```bash
TOKEN=$(login userA)   # MFA NOT completed

curl -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"amount": 50000, "to": "IBAN"}' \
  https://api.mavibank.local/v1/transfers

→ 401 / step_up_required + challenge_id
```

#### Senaryo 3: JWT tampering

```bash
TOKEN=$(login userA)
# Modify payload to set role=admin
TAMPERED=$(modify_jwt_payload $TOKEN '.realm_access.roles += ["admin"]')

curl -H "Authorization: Bearer $TAMPERED" \
  https://api.mavibank.local/admin/users

→ 401 Invalid signature (RSA verification fails)
```

#### Senaryo 4: Brute force lockout

```bash
for i in {1..6}; do
  curl -X POST -d "username=ahmet&password=wrong$i" \
    https://auth.mavibank.local/realms/banking/protocol/openid-connect/token
done

→ 5. attempt onwards: invalid_grant
→ 6. attempt: account locked
→ Audit log: 5 LOGIN_FAIL + USER_LOCKED
→ SecurityOps alert
```

#### Senaryo 5: Refresh token reuse

```bash
TOKENS_1=$(login userA)
REFRESH=$(echo $TOKENS_1 | jq -r '.refresh_token')

# First use - OK
TOKENS_2=$(refresh $REFRESH)

# Second use - COMPROMISE
TOKENS_3=$(refresh $REFRESH)

→ All user A tokens revoked
→ Audit: TOKEN_REUSE_DETECTED
→ SecurityOps alert
→ User notification SMS
```

#### Senaryo 6: SQL injection attempt

```bash
curl -X POST -d "username=admin' OR '1'='1&password=anything" \
  https://api.mavibank.local/v1/auth/login

→ 401 Unauthorized
→ Audit: LOGIN_FAIL username=admin (sanitized — not the SQL fragment)
→ WAF flag (if enabled)
```

#### Senaryo 7: SSRF attempt (notification service URL preview)

```bash
curl -X POST -H "Authorization: Bearer $TOKEN" \
  -d "url=http://169.254.169.254/latest/meta-data/" \
  https://api.mavibank.local/v1/notifications/preview

→ 403 Forbidden — Host not in allowlist
→ Audit: SSRF_ATTEMPT user=X, url=http://169.254.169.254/...
→ SecurityOps alert
```

#### Senaryo 8: Audit trail tampering

```bash
# Direct DB access (simulating insider threat)
psql -c "UPDATE audit_event SET action='FAKE' WHERE id=12345"

# Verify chain
curl https://api.mavibank.local/audit/verify-chain

→ {"valid": false, "first_mismatch_id": 12345, ...}
→ Tamper detected, all subsequent rows flagged
→ SecurityOps alert
```

### 11. Defter notları (15 item)

1. "Keycloak realm banking + 6 client (mobile, web, api, teller, payment-svc, notification-svc) tasarımı: ____."
2. "JWT propagation gateway → 4 service + her servisin kendi validation: ____."
3. "Banking MFA flow (TOTP + risk SPI + WebAuthn yüksek değer): ____."
4. "Encryption stack: AES-GCM + envelope + KMS (LocalStack/AWS) + blind index: ____."
5. "PAN tokenization (PCI scope reduction) + audited detokenize: ____."
6. "OWASP A01-A10 banking-specific defenses listesi: ____."
7. "Audit hash chain (tamper-proof) + Kafka topic + verify scheduled: ____."
8. "Step-up authentication (yüksek tutar OTP challenge + Redis): ____."
9. "Spring Security 6 + Keycloak + JwtAuthenticationConverter realm/client role mapping: ____."
10. "Logging masking (TC, PAN, Authorization header) + ELK indexing: ____."
11. "Security headers (HSTS, CSP, XFO, XCT, Referrer-Policy) banking baseline: ____."
12. "OWASP dependency-check + SBOM + Trivy CI integration: ____."
13. "Refresh token rotation + compromise detection + tokens revoke: ____."
14. "Kasten kırma 8 senaryosu (IDOR, step-up bypass, JWT tampering, brute force, refresh reuse, SQLi, SSRF, audit tamper) — all reproducible + fixed: ____."
15. "BDDK + KVKK + PCI-DSS compliance mapping (encryption, audit retention 5-10 yıl, MFA, tokenization, access control): ____."

---

## Tamamlama kriterleri

- [ ] Keycloak realm + 6 client + 5 role + custom MFA flow
- [ ] 4 service + Spring Security 6 + JWT validation + JwtAuthenticationConverter
- [ ] Gateway TokenRelay (JWT propagation) + per-service revalidation
- [ ] Encryption: AES-GCM + envelope + KMS (LocalStack) + 3 service customers/cards/transfers
- [ ] PAN tokenization + audited detokenize
- [ ] Audit service + hash chain + Kafka topic + 14 audit event type
- [ ] Step-up authentication (>10k TL, international, new beneficiary)
- [ ] Logging masking (TC, PAN, Bearer)
- [ ] Security headers (HSTS, CSP, XFO, XCT, Referrer-Policy)
- [ ] OWASP dependency-check CI + Trivy + SpotBugs FindSecBugs
- [ ] 25+ integration test (her OWASP A için min 1)
- [ ] OWASP ZAP baseline scan report
- [ ] 8 kasten kırma senaryosu reproduce + fix + audit verify
- [ ] 15 defter notu
- [ ] Compliance mapping doc (KVKK + BDDK + PCI-DSS)

---

## Önemli not — sıralama

1. Keycloak setup → realm + client
2. Bir service (account) → Spring Security + JWT
3. Diğer 3 service → aynı pattern
4. Gateway TokenRelay
5. Encryption (column-level + tokenization)
6. Audit service + Kafka
7. Step-up flow
8. OWASP hardening (security headers, masking, etc.)
9. Testing (integration + DAST + SAST)
10. Kasten kırma senaryoları + fix + verify

Her adım'da **kasten kırma + reproduce** done. Demo videosu için 8 senaryo capture.
