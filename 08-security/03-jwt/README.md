# Topic 8.3 — JWT: JWS, JWE, Refresh Token Rotation

## Hedef

JWT (JSON Web Token) yapısını banking-grade derinlikte öğrenmek. JWS (signed) vs JWE (encrypted), HMAC vs RSA vs ECDSA algoritmaları, access/refresh token pattern, refresh token rotation, token revocation (blocklist), security pitfalls ve fixes. Spring Security 6 + Spring Authorization Server entegrasyonu.

## Süre

Okuma: 2 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- Topic 8.2 (Authentication) bitti
- Phase 7 (API Gateway) — JWT validation gateway tarafında gördük
- Public/private key cryptography temel

---

## Kavramlar

### 1. JWT — yapı

JWT = three dot-separated base64url parts:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyLTEyMyIsImlhdCI6MTcxNjAwMDAwMH0.SflKxwRJSMeKKF2QT4f...
   header           payload                                         signature
```

**Header** (base64):

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "key-id-1"
}
```

**Payload (claims)** (base64):

```json
{
  "iss": "https://auth.mavibank.com",
  "sub": "user-123",
  "aud": "banking-api",
  "exp": 1716003600,
  "iat": 1716000000,
  "jti": "uuid-token-id",
  "scope": "account.read transfer.write",
  "roles": ["customer"],
  "tenant": "tr"
}
```

**Signature** (base64):

HMAC veya RSA — header + payload imzalanır.

### 2. Standard claims (RFC 7519)

| Claim | Anlam |
|---|---|
| `iss` (issuer) | Token issuer (auth server) |
| `sub` (subject) | User ID veya principal |
| `aud` (audience) | Intended recipient (API) |
| `exp` (expiration) | Expiration timestamp (epoch seconds) |
| `iat` (issued at) | Token issue time |
| `nbf` (not before) | Token valid from |
| `jti` (JWT ID) | Unique token ID (revocation için) |

**Banking custom claims:**

```json
{
  "tenant": "tr",
  "branch": "istanbul-1",
  "customer_id": "cust-456",
  "permissions": ["transfer.read", "transfer.write"],
  "session_id": "sess-789",
  "mfa_completed": true,
  "ip": "192.168.1.5",
  "device_id": "device-abc"
}
```

**Tehlike:** JWT payload **base64**, **encrypted DEĞİL**. Herkes decode edip okuyabilir. **PII KOYMA.**

### 3. JWS vs JWE

#### JWS (JSON Web Signature) — yaygın

Token **signed** ama **content readable** (base64).

```
header.payload.signature
       ↑
    base64 decode = JSON plain
```

Tampering tespit edilir (signature mismatch), ama content gizli değil.

#### JWE (JSON Web Encryption) — content encrypted

Token **encrypted**. Sadece authorized party decrypt eder.

```
header.encrypted_key.iv.ciphertext.authentication_tag
```

5 parça. AES-GCM gibi simetrik şifre + asymmetric key exchange.

**Banking pratiği:** Çoğunlukla **JWS** + payload'da PII koyma. PII gerekirse **DB lookup** (token sadece reference).

Banking PII'yi JWE içinde tutmak yerine, JWT'ye sadece `userId` koy, detayı API'den iste.

### 4. Algorithms

| Algorithm | Tür | Banking Adoption |
|---|---|---|
| **HS256** | HMAC SHA-256, symmetric | Single-service (issuer = verifier same secret) |
| **HS384, HS512** | HMAC larger | Aynı, daha sıkı |
| **RS256** | RSA SHA-256, asymmetric | ✓ Microservice (banking standard) |
| **RS384, RS512** | RSA larger | OK |
| **ES256** | ECDSA P-256 SHA-256 | Daha kısa key, modern |
| **PS256** | RSA-PSS | RSA varyasyonu |

#### HS256 — symmetric key

Issuer + verifier **aynı secret**.

```yaml
jwt:
  secret: "verylongsecret_at_least_32_bytes_for_HS256"
```

**Tehlike:** Secret leak = herkes geçerli token yaratabilir. Multi-service distribution'da hayli risk.

#### RS256 — asymmetric

Issuer **private key** ile imzalar. Verifier'lar **public key** ile doğrular.

```
[Auth Server] - private key (gizli)
   ↓ issues
   token signed with private key
   ↓
[Resource Server 1] - public key
[Resource Server 2] - public key
[Resource Server 3] - public key
```

Public key herkesle paylaşılabilir. Private key sadece auth server'da.

**Banking standard:** RS256.

#### Algorithm confusion attack

```
Attacker:
  Headers: {"alg": "HS256"}   (was RS256)
  Try to verify with HMAC using public key as secret
```

Naive verifier: "alg=HS256 specified, use HMAC". Public key (already public) used as HMAC secret → forged token validates.

**Fix:** **Explicit algorithm enforcement** — verifier kabul edilen algoritmaları sabitler.

```java
JwtParser parser = Jwts.parser()
    .verifyWith(publicKey)
    .build();
// Spring Security oauth2 resource server bunu doğru yapar
```

### 5. `alg: none` attack

```json
{"alg": "none", "typ": "JWT"}
```

Naive verifier: "alg=none → signature skip". **Token fake, verifier accept eder.**

Modern library'lerde fixed. Banking için **manuel kontrol**:

```java
if ("none".equalsIgnoreCase(jwsHeader.getAlgorithm().getName())) {
    throw new SecurityException("Algorithm 'none' is not allowed");
}
```

### 6. Spring Security 6 — Resource Server JWT

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.mavibank.com/realms/banking
          # Otomatik JWK set fetch
          jwk-set-uri: https://auth.mavibank.com/realms/banking/protocol/openid-connect/certs
```

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class BankingResourceServerConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(c -> c.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(a -> a
                .requestMatchers("/v1/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("admin")
                .requestMatchers("/v1/transfers/**").hasAuthority("SCOPE_transfer.write")
                .anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt(j -> j.jwtAuthenticationConverter(jwtConverter())));
        return http.build();
    }
    
    @Bean
    public JwtAuthenticationConverter jwtConverter() {
        JwtGrantedAuthoritiesConverter authConverter = new JwtGrantedAuthoritiesConverter();
        authConverter.setAuthorityPrefix("ROLE_");
        authConverter.setAuthoritiesClaimName("roles");
        
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            // Custom: realm_access.roles + scope
            Set<GrantedAuthority> authorities = new HashSet<>();
            
            // Roles
            Map<String, Object> realmAccess = jwt.getClaim("realm_access");
            if (realmAccess != null) {
                @SuppressWarnings("unchecked")
                List<String> roles = (List<String>) realmAccess.get("roles");
                if (roles != null) {
                    roles.forEach(r -> authorities.add(new SimpleGrantedAuthority("ROLE_" + r)));
                }
            }
            
            // Scopes
            String scope = jwt.getClaimAsString("scope");
            if (scope != null) {
                for (String s : scope.split(" ")) {
                    authorities.add(new SimpleGrantedAuthority("SCOPE_" + s));
                }
            }
            
            return authorities;
        });
        converter.setPrincipalClaimName("sub");
        return converter;
    }
}
```

Spring otomatik:
1. Issuer URI'den `/.well-known/openid-configuration` fetch
2. JWK set URI'den public key'leri çek (rotation'a otomatik adapt)
3. Her gelen JWT'i validate (signature + claims)

Controller:

```java
@RestController
@RequestMapping("/v1")
public class TransferController {
    
    @PostMapping("/transfers")
    @PreAuthorize("hasAuthority('SCOPE_transfer.write')")
    public Mono<Transfer> transfer(
        @RequestBody TransferRequest req,
        @AuthenticationPrincipal Jwt jwt
    ) {
        UUID userId = UUID.fromString(jwt.getSubject());
        String tenant = jwt.getClaimAsString("tenant");
        // ...
    }
}
```

### 7. Access + Refresh token pattern

**Access token:** Kısa ömürlü (5-15 dk). Her API request'inde kullanılır.

**Refresh token:** Uzun ömürlü (1-24 saat). Sadece yeni access token almak için.

```
Login → access (15 min) + refresh (1 hour)

Time 0: Client uses access
Time 14m: Access geçerli
Time 16m: Access expired → 401
  Client: POST /auth/refresh with refresh token
  Server: returns new access + new refresh
Time 1h+: Refresh expired → re-login required
```

**Banking pratiği:**
- Access TTL: 5-15 dakika
- Refresh TTL: 1-24 saat (banking için **kısa** — sensitive)
- Refresh **rotation**: her refresh'te yeni refresh token

### 8. Refresh token rotation — detaylı

```
1. Login → A1 (access) + R1 (refresh)
2. A1 expired → POST /refresh (R1)
3. Server: 
   - Validate R1
   - Mark R1 as USED (one-time use)
   - Issue A2 + R2
   - Return (A2, R2)
4. Time later, A2 expired → POST /refresh (R2)
5. Cycle continues
```

**Compromise detection:**

```
Attacker steals R1.
Real user uses R1 → success, get R2.
Attacker tries R1 → SERVER: "R1 was already used! COMPROMISE."
  - Revoke ALL refresh tokens for user
  - Force re-login
  - Alert security ops
```

```java
@Service
public class RefreshTokenService {
    
    private final RefreshTokenRepository repo;
    
    @Transactional
    public TokenPair refresh(String refreshToken) {
        DecodedJWT decoded = verifyJwt(refreshToken);
        UUID tokenId = UUID.fromString(decoded.getId());
        UUID userId = UUID.fromString(decoded.getSubject());
        
        RefreshTokenRecord record = repo.findById(tokenId)
            .orElseThrow(() -> new InvalidTokenException("Token not found"));
        
        if (record.isUsed()) {
            // REUSE DETECTION → compromise
            log.error("Refresh token reuse detected: user={}, token={}", userId, tokenId);
            repo.revokeAllForUser(userId);
            alerts.notifySecurityOps(
                "Possible refresh token theft for user: " + userId);
            
            // Notify user
            notificationService.sendSecurityAlert(userId, 
                "Suspicious activity detected. Please re-login.");
            
            throw new SecurityException("Refresh token reuse detected");
        }
        
        if (record.isRevoked() || record.getExpiresAt().isBefore(Instant.now())) {
            throw new InvalidTokenException("Token revoked or expired");
        }
        
        // Mark as used
        record.setUsed(true);
        record.setUsedAt(Instant.now());
        repo.save(record);
        
        // Issue new tokens
        return issueTokenPair(userId);
    }
}
```

**Banking standard:** Refresh rotation + compromise detection **şart**.

### 9. Token revocation

Stateless JWT'nin zayıflığı: signature valid olduğu sürece token geçerli. Logout veya security incident'ta nasıl iptal?

#### Yöntem 1: Blocklist (Redis)

```java
@Service
public class TokenBlocklistService {
    
    private final RedisTemplate<String, String> redis;
    
    public void revoke(String jti, long expiresIn) {
        redis.opsForValue().set(
            "blocked:" + jti, 
            "1", 
            Duration.ofSeconds(expiresIn)
        );
        // TTL = token expiry → cleanup otomatik
    }
    
    public boolean isRevoked(String jti) {
        return redis.hasKey("blocked:" + jti);
    }
}
```

Resource server her request'te blocklist check:

```java
@Component
public class JwtBlocklistFilter implements WebFilter {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        return ReactiveSecurityContextHolder.getContext()
            .flatMap(ctx -> {
                Authentication auth = ctx.getAuthentication();
                if (auth instanceof JwtAuthenticationToken jwtAuth) {
                    String jti = jwtAuth.getToken().getId();
                    if (blocklistService.isRevoked(jti)) {
                        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                        return exchange.getResponse().setComplete();
                    }
                }
                return chain.filter(exchange);
            })
            .switchIfEmpty(chain.filter(exchange));
    }
}
```

**Trade-off:** Her request Redis lookup → ~1ms latency. Banking için acceptable.

#### Yöntem 2: Short-lived + refresh revoke

Access token **5 dakika**. Compromise tespit edilirse:
- Refresh token revoke (immediate)
- Max 5 dakika sonra access da expired

Banking pratiği: Blocklist + short-lived combo.

#### Yöntem 3: Token versioning

User'da `tokenVersion` field. Her token'a current version embed. Logout → version++. Eski version'lı token'lar invalid.

```java
public class User {
    ...
    private int tokenVersion;   // increment on logout
}

// Token issuance
JWT.create()
    .withClaim("token_version", user.getTokenVersion())
    ...

// Validation
int tokenVersion = jwt.getClaim("token_version").asInt();
if (tokenVersion != user.getTokenVersion()) {
    throw new InvalidTokenException();
}
```

Simpler ama DB lookup her request'te.

### 10. Banking implementation — TokenService

```java
@Service
public class TokenService {
    
    @Value("${jwt.private-key-path}")
    private String privateKeyPath;
    
    @Value("${jwt.public-key-path}")
    private String publicKeyPath;
    
    private RSAPrivateKey privateKey;
    private RSAPublicKey publicKey;
    
    @PostConstruct
    public void init() {
        this.privateKey = loadPrivateKey(privateKeyPath);
        this.publicKey = loadPublicKey(publicKeyPath);
    }
    
    public TokenPair issue(User user) {
        Instant now = Instant.now();
        String sessionId = UUID.randomUUID().toString();
        
        String accessToken = JWT.create()
            .withIssuer("https://auth.mavibank.com")
            .withSubject(user.getId().toString())
            .withAudience("banking-api")
            .withIssuedAt(now)
            .withExpiresAt(now.plus(15, ChronoUnit.MINUTES))
            .withJWTId(UUID.randomUUID().toString())
            .withClaim("tenant", user.getTenant())
            .withClaim("session_id", sessionId)
            .withClaim("mfa_completed", user.isMfaCompleted())
            .withArrayClaim("roles", user.getRoles().toArray(new String[0]))
            .withClaim("scope", String.join(" ", user.getScopes()))
            .sign(Algorithm.RSA256(publicKey, privateKey));
        
        UUID refreshTokenId = UUID.randomUUID();
        String refreshToken = JWT.create()
            .withIssuer("https://auth.mavibank.com")
            .withSubject(user.getId().toString())
            .withIssuedAt(now)
            .withExpiresAt(now.plus(1, ChronoUnit.HOURS))
            .withJWTId(refreshTokenId.toString())
            .withClaim("token_type", "refresh")
            .withClaim("session_id", sessionId)
            .sign(Algorithm.RSA256(publicKey, privateKey));
        
        // Store refresh token record
        refreshTokenRepo.save(RefreshTokenRecord.builder()
            .id(refreshTokenId)
            .userId(user.getId())
            .sessionId(UUID.fromString(sessionId))
            .issuedAt(now)
            .expiresAt(now.plus(1, ChronoUnit.HOURS))
            .used(false)
            .revoked(false)
            .build());
        
        return new TokenPair(accessToken, refreshToken);
    }
    
    public TokenPair refresh(String refreshToken) {
        DecodedJWT decoded = JWT.require(Algorithm.RSA256(publicKey, null))
            .withIssuer("https://auth.mavibank.com")
            .withClaim("token_type", "refresh")
            .build()
            .verify(refreshToken);
        
        UUID tokenId = UUID.fromString(decoded.getId());
        UUID userId = UUID.fromString(decoded.getSubject());
        
        RefreshTokenRecord record = refreshTokenRepo.findById(tokenId)
            .orElseThrow(() -> new InvalidTokenException("Token not found"));
        
        if (record.isUsed()) {
            // COMPROMISE
            log.error("Refresh token reuse: user={}", userId);
            refreshTokenRepo.revokeAllForUser(userId);
            alerts.notifySecurityOps("Refresh token reuse: " + userId);
            throw new SecurityException("Token reuse detected");
        }
        
        if (record.isRevoked()) {
            throw new InvalidTokenException("Token revoked");
        }
        
        // Mark as used
        record.setUsed(true);
        record.setUsedAt(Instant.now());
        refreshTokenRepo.save(record);
        
        // Issue new pair
        User user = userRepo.findById(userId).orElseThrow();
        return issue(user);
    }
    
    public void revoke(String accessToken) {
        DecodedJWT decoded = JWT.decode(accessToken);
        String jti = decoded.getId();
        long expiresIn = decoded.getExpiresAt().toInstant().getEpochSecond() 
                        - Instant.now().getEpochSecond();
        if (expiresIn > 0) {
            blocklistService.revoke(jti, expiresIn);
        }
    }
}
```

### 11. JWKS — public key rotation

Auth server periodic key rotation. JWKS endpoint:

```
GET /.well-known/jwks.json

{
  "keys": [
    {
      "kty": "RSA",
      "kid": "key-2025-q1",
      "alg": "RS256",
      "use": "sig",
      "n": "...",   // modulus
      "e": "AQAB"   // exponent
    },
    {
      "kid": "key-2025-q2",
      ...
    }
  ]
}
```

Token header'da `kid` → verifier o key ile validate.

Spring Security `jwk-set-uri` ile otomatik fetch + cache (5 dk refresh).

Banking pratiği: Key rotation **3-12 ay**. Compromise durumunda anlık.

### 12. JWT security pitfalls — banking

#### Pitfall 1: `alg: none`

Yukarıda anlatıldı.

#### Pitfall 2: Algorithm confusion

Yukarıda. **Explicit algorithm enforcement.**

#### Pitfall 3: Weak HS256 secret

```yaml
jwt:
  secret: "secret"   # ❌ guess-able
```

Min 256-bit random key:

```bash
openssl rand -base64 32
# → "Random32ByteString..."
```

HS256 minimum **32-byte** secret.

#### Pitfall 4: No `exp` claim

Sonsuza kadar geçerli token. **Her token `exp` set olmalı.**

#### Pitfall 5: Sensitive data in payload

```json
{
  "sub": "user-123",
  "tc_kimlik": "12345678901",       // ❌
  "card_pan": "4111-1111-1111-1234", // ❌
  "balance": "150000.00"             // ❌
}
```

Base64 decode = plaintext görünür. **PII koyma.**

Banking: Sadece internal ID (UUID), role, scope.

#### Pitfall 6: No `aud` (audience) check

Auth server multiple API için token issue ediyor:

```
auth.banking.com issues token with aud="banking-api"
```

Bu token başka API'ye gönderiliyor:

```
shopping.banking.com: receives token with aud="banking-api"
```

Naive verifier: signature valid → accept. **Banking-api token shopping API'sinde kullanıldı.**

**Fix:** Resource server `aud` claim check.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.mavibank.com
          audiences: ["banking-api"]
```

#### Pitfall 7: Clock skew

Server A: time = 10:00:00
Server B: time = 10:00:30

A issues token with exp=10:15:00.
At 10:14:45 (real), B receives token. B's clock = 10:15:15 → exp passed → reject.

**Fix:** NTP synchronization. Verifier clock skew tolerance (`leeway` parameter).

```java
JWT.require(...)
    .acceptLeeway(5)   // 5 sn tolerance
    .build();
```

Banking pratiği: NTP cluster-wide. Leeway 5-30 saniye.

### 13. JWE — when needed

Banking için JWE nadir:
- Highly sensitive payload (özel kart bilgisi)
- Inter-bank communication
- B2B Open Banking

```java
RSAEncrypter encrypter = new RSAEncrypter(publicKey);

JWEHeader header = new JWEHeader(JWEAlgorithm.RSA_OAEP_256, EncryptionMethod.A256GCM);
JWTClaimsSet claims = new JWTClaimsSet.Builder()
    .subject("user-123")
    .claim("sensitive_data", "...")
    .build();

EncryptedJWT jwt = new EncryptedJWT(header, claims);
jwt.encrypt(encrypter);

String serialized = jwt.serialize();   // 5-part JWE
```

Decryption: private key.

**Trade-off:** Bigger token, slower processing. JWS + DB lookup genelde daha pragmatik.

### 14. Banking anti-pattern'leri

**Anti-pattern 1: HS256 microservice'lerde**

Symmetric secret tüm servislere paylaşılır. Bir servis compromise → tüm token'lar fake'lenebilir.

**Banking standard:** RS256 (asymmetric).

**Anti-pattern 2: HS256 hardcoded weak secret**

```yaml
jwt.secret: "secret"
```

Guess-able. Min 256-bit random.

**Anti-pattern 3: Sensitive data payload'da**

JWT base64. PII = leak.

**Anti-pattern 4: Refresh rotation yok**

Refresh stealed → forever access. Rotation şart.

**Anti-pattern 5: Compromise detection yok**

Reuse → silent re-issue. Saldırı görünmez.

**Anti-pattern 6: Token TTL çok uzun**

```
Access TTL: 24 hours
```

Stealed token 24 saat valid. **Short access (5-15 dk) + refresh** pattern.

**Anti-pattern 7: alg=none accept**

Modern library'lerde fixed ama config'le açılabilir. Manuel ban list.

**Anti-pattern 8: aud check yok**

Cross-service token reuse.

**Anti-pattern 9: Logout sadece client-side**

```javascript
localStorage.removeItem('token');   // ❌ token Server-side hala valid
```

Server-side revocation (blocklist) şart.

**Anti-pattern 10: Refresh token Authorization header'da**

```
Authorization: Bearer <refresh_token>
```

Refresh token kullanım scope'unu kısıtla: sadece `/auth/refresh` endpoint, ayrı cookie/body.

---

## Önemli olabilecek araştırma kaynakları

- RFC 7519 — JWT specification
- RFC 7515 — JWS
- RFC 7516 — JWE
- RFC 7517 — JWK / JWKS
- OWASP JWT Cheat Sheet
- Auth0 blog — JWT best practices
- Spring Security OAuth2 reference

---

## Mini task'ler

### Task 8.3.1 — RSA key pair generation (15 dk)

```bash
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

`private.pem`, `public.pem` Spring Boot resource'a koy. **Production: vault'tan al, code'a yazma.**

### Task 8.3.2 — TokenService RS256 issuance (60 dk)

Yukarıdaki `TokenService.issue()` implement. Access (15 min) + Refresh (1h) pair.

Test: Token decode (jwt.io) → claims doğru görünmeli.

### Task 8.3.3 — Refresh token rotation + DB tracking (60 dk)

`RefreshTokenRecord` entity + repo. Refresh → mark as used, issue new pair.

Test: 3 refresh sıralı yap, her seferinde yeni R verilmeli.

### Task 8.3.4 — Compromise detection (45 dk)

Used refresh token tekrar kullanıldığında: `revokeAllForUser` + alert.

Test:
1. R1 kullan → R2 al
2. R1 tekrar kullan → SecurityException + tüm token'lar revoke

### Task 8.3.5 — Spring Security resource server JWT (60 dk)

`oauth2ResourceServer.jwt()` config. JwtAuthenticationConverter ile role + scope mapping.

Test:
- Valid JWT → endpoint reach
- Expired JWT → 401
- Wrong audience → 401
- Insufficient role → 403

### Task 8.3.6 — Token blocklist Redis (45 dk)

`TokenBlocklistService`. Logout → blocklist + TTL.

Resource server filter blocklist check.

Test: Logout → token hala geçerli olmamalı (blocked).

### Task 8.3.7 — JWT claim validation (30 dk)

Verifier: `iss`, `aud`, `exp` check. Wrong issuer / audience reject.

### Task 8.3.8 — Algorithm confusion test (30 dk)

Saldırgan HS256 token public key ile imzalamaya çalışsın. Reject olmalı.

---

## Test yazma rehberi

```java
@SpringBootTest
class TokenServiceTest {
    
    @Autowired TokenService tokenService;
    @Autowired RefreshTokenRepository repo;
    
    @Test
    void shouldIssueValidTokenPair() {
        User user = createTestUser();
        
        TokenPair pair = tokenService.issue(user);
        
        DecodedJWT access = JWT.decode(pair.accessToken());
        assertThat(access.getSubject()).isEqualTo(user.getId().toString());
        assertThat(access.getIssuer()).isEqualTo("https://auth.mavibank.com");
        assertThat(access.getAudience()).contains("banking-api");
        assertThat(access.getExpiresAt())
            .isCloseTo(Date.from(Instant.now().plus(15, ChronoUnit.MINUTES)), 
                within(5, TimeUnit.SECONDS));
    }
    
    @Test
    void shouldRotateRefreshToken() {
        TokenPair pair1 = tokenService.issue(createTestUser());
        
        TokenPair pair2 = tokenService.refresh(pair1.refreshToken());
        
        assertThat(pair2.refreshToken()).isNotEqualTo(pair1.refreshToken());
        
        // Old refresh now USED
        UUID oldTokenId = UUID.fromString(JWT.decode(pair1.refreshToken()).getId());
        RefreshTokenRecord oldRecord = repo.findById(oldTokenId).orElseThrow();
        assertThat(oldRecord.isUsed()).isTrue();
    }
    
    @Test
    void shouldDetectRefreshTokenReuse() {
        User user = createTestUser();
        TokenPair pair1 = tokenService.issue(user);
        
        TokenPair pair2 = tokenService.refresh(pair1.refreshToken());
        
        // Try to use pair1.refreshToken again
        assertThatThrownBy(() -> tokenService.refresh(pair1.refreshToken()))
            .isInstanceOf(SecurityException.class);
        
        // All user's refresh tokens revoked
        UUID pair2TokenId = UUID.fromString(JWT.decode(pair2.refreshToken()).getId());
        RefreshTokenRecord pair2Record = repo.findById(pair2TokenId).orElseThrow();
        assertThat(pair2Record.isRevoked()).isTrue();
    }
    
    @Test
    void shouldRejectExpiredAccessToken() throws Exception {
        // Mock token with past expiry
        String expired = JWT.create()
            .withSubject("user-123")
            .withExpiresAt(Date.from(Instant.now().minus(1, ChronoUnit.HOURS)))
            .sign(Algorithm.RSA256(publicKey, privateKey));
        
        mockMvc.perform(get("/v1/accounts/123")
            .header("Authorization", "Bearer " + expired))
            .andExpect(status().isUnauthorized());
    }
    
    @Test
    void shouldRejectWrongAudience() throws Exception {
        String wrongAud = JWT.create()
            .withSubject("user-123")
            .withAudience("wrong-api")   // banking-api değil
            .withExpiresAt(Date.from(Instant.now().plus(15, ChronoUnit.MINUTES)))
            .sign(Algorithm.RSA256(publicKey, privateKey));
        
        mockMvc.perform(get("/v1/accounts/123")
            .header("Authorization", "Bearer " + wrongAud))
            .andExpect(status().isUnauthorized());
    }
    
    @Test
    void shouldNotIncludeSensitiveDataInPayload() {
        User user = User.builder()
            .id(UUID.randomUUID())
            .tcKimlik("12345678901")
            .cardPan("4111-1111-1111-1234")
            .build();
        
        TokenPair pair = tokenService.issue(user);
        
        DecodedJWT decoded = JWT.decode(pair.accessToken());
        Map<String, Claim> claims = decoded.getClaims();
        
        assertThat(claims.toString()).doesNotContain("12345678901");
        assertThat(claims.toString()).doesNotContain("4111-1111-1111");
    }
}
```

---

## Claude-verify prompt

```
JWT implementation'ımı banking-grade kriterlere göre değerlendir:

1. Algorithm:
   - RS256 (asymmetric) microservice'lerde?
   - HS256 hardcoded weak secret YOK?
   - Explicit algorithm enforcement (alg confusion prevention)?
   - alg=none accept etmiyor mu?

2. Standard claims:
   - exp claim her token'da?
   - iss, aud, sub explicit?
   - jti unique (revocation için)?
   - Clock skew leeway (5-30 sn)?

3. Payload:
   - PII (TC No, card PAN, balance) payload'da YOK?
   - Sadece internal ID (UUID), role, scope?

4. Access + Refresh pattern:
   - Access TTL 5-15 dk?
   - Refresh TTL 1-24 saat (banking için kısa)?
   - Refresh rotation (her refresh'te yeni R)?

5. Compromise detection:
   - Used refresh token reuse → SecurityException?
   - All user tokens revoke?
   - Security ops alert?

6. Token revocation:
   - Blocklist (Redis) ile logout?
   - TTL token expiry'sıyla aynı (cleanup)?
   - Resource server filter blocklist check?

7. Spring Security:
   - oauth2ResourceServer JWT setup?
   - JwtAuthenticationConverter realm_access + scope mapping?
   - Stateless session?

8. JWKS:
   - Auth server JWKS endpoint?
   - Resource server jwk-set-uri auto-fetch?
   - Key rotation strategy?

9. Banking-specific:
   - Refresh token /auth/refresh endpoint'inden başka yerde kullanılmıyor?
   - Audience check production'da?
   - Custom claim (tenant, session_id) banking için?

10. Anti-pattern:
    - HS256 microservice'te?
    - Weak secret?
    - Long access TTL (24+ saat)?
    - Refresh rotation yok?
    - Sensitive payload?
    - alg=none accept?
    - aud check yok?
    - Client-side only logout?

Her madde için PASS / FAIL / EKSIK işaretle.
```

---

## Tamamlama kriterleri

- [ ] RSA 2048 key pair generated
- [ ] TokenService.issue() RS256 + 15 min access + 1h refresh
- [ ] Refresh rotation + DB tracking
- [ ] Compromise detection (reuse → revoke all)
- [ ] Spring Security resource server JWT validation
- [ ] JwtAuthenticationConverter role + scope
- [ ] Token blocklist Redis (logout)
- [ ] Claim validation (iss, aud, exp)
- [ ] Algorithm confusion test
- [ ] 6+ integration test
- [ ] PII payload'da YOK kontrolü

---

## Defter notları (10 madde)

1. "JWT 3 parça (header.payload.signature) + base64 decode anlamı: ____."
2. "JWS vs JWE ne zaman hangisi: ____."
3. "HS256 vs RS256 microservice senaryosu: ____."
4. "Algorithm confusion attack + explicit enforcement fix: ____."
5. "Access + Refresh pattern banking TTL kararı: ____."
6. "Refresh rotation + compromise detection mekanizması: ____."
7. "Token revocation blocklist (Redis + TTL) banking: ____."
8. "aud claim cross-service token reuse prevention: ____."
9. "JWT payload PII koyma anti-pattern (KVKK + PCI-DSS): ____."
10. "JWKS endpoint key rotation auth server tarafı: ____."
