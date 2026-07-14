# Topic 8.3 — JWT: JWS, JWE, Refresh Token Rotation

```admonish info title="Bu bölümde"
- JWT'nin üç parçası (header.payload.signature) ve neden payload'ın "şifreli değil, sadece imzalı" olduğu
- HMAC (HS256) vs RSA (RS256) vs ECDSA — microservice'te hangisi ve `alg=none` / algorithm confusion saldırıları
- Access + refresh token pattern, refresh token rotation ve reuse ile compromise detection
- Stateless JWT'yi geri alma (revocation): Redis blocklist, short-lived + refresh revoke, token versioning
- Banking pitfall cephaneliği: weak secret, PII payload, eksik `aud`/`exp`, clock skew, cross-service token reuse
```

## Hedef

JWT (JSON Web Token) yapısını banking-grade derinlikte kavramak. JWS (signed) vs JWE (encrypted) ayrımını, HMAC/RSA/ECDSA algoritmalarını ve bunların saldırı yüzeyini (`alg=none`, algorithm confusion) hatasız anlatabilmek. Access/refresh token pattern'ini, refresh rotation + compromise detection'ı ve stateless token'ı revocation stratejilerini tasarlayabilmek. Tüm bunları Spring Security 6 + OAuth2 Resource Server ile pratiğe dökebilmek.

## Süre

Okuma: ~2 saat • Kendini Sına: 45 dk • Pratik (opsiyonel): 3-4 saat • Toplam: ~2.5 saat (+ pratik)

## Önbilgi

- Topic 8.2 (Authentication) bitti — kimlik doğrulama akışını biliyorsun
- Phase 7 (API Gateway) — JWT validation'ı gateway tarafında gördün
- Public/private key cryptography temel seviyede oturmuş (asymmetric imza fikri)

---

## Kavramlar

### 1. Stateless auth ve JWT yapısı

Neden önemli: microservice mimaride her request için server-side session tutmak ölçeklenmez — 10 servis aynı session store'a bakmak zorunda kalır. **Stateless authentication** bu bağı koparır: kimlik bilgisini imzalı bir token içinde taşırsın, her servis token'ı kendi başına doğrular, ortak state gerekmez.

**JWT (JSON Web Token)** bunun standart formatıdır: nokta ile ayrılmış üç base64url parça.

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyLTEyMyIsImlhdCI6MTcxNjAwMDAwMH0.SflKxwRJSMeKKF2QT4f...
   header           payload                                         signature
```

Üç parçanın işlevi ayrıdır: header algoritmayı söyler, payload iddiaları (claims) taşır, signature ilk ikisini kriptografik olarak mühürler.

```mermaid
flowchart LR
    T["JWT string"] --> H["Header base64url"]
    T --> P["Payload base64url"]
    T --> S["Signature base64url"]
    H --> HC["alg typ kid"]
    P --> PC["iss sub aud exp claims"]
    S --> SC["Header ve Payload imzasi"]
```

**Header** algoritma ve anahtar kimliğini (`kid`) belirtir:

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "key-id-1"
}
```

**Payload (claims)** asıl kimlik ve yetki bilgisidir:

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

**Signature** ise `base64url(header) + "." + base64url(payload)` üzerinden HMAC veya RSA ile üretilir — tampering'i (payload'ı değiştirme) yakalayan tek mekanizma budur.

### 2. Standard claims ve payload'a ne konur

Neden önemli: claim'lerin bir kısmı RFC 7519'da standarttır ve doğrulama bu isimlere dayanır (`exp`, `aud`, `iss`); yanlış isimlendirirsen kütüphane doğrulamayı atlar.

| Claim | Anlam |
|---|---|
| `iss` (issuer) | Token'ı üreten auth server |
| `sub` (subject) | User ID veya principal |
| `aud` (audience) | Token'ın hedef alıcısı (API) |
| `exp` (expiration) | Bitiş zamanı (epoch seconds) |
| `iat` (issued at) | Üretim zamanı |
| `nbf` (not before) | Geçerlilik başlangıcı |
| `jti` (JWT ID) | Benzersiz token ID (revocation için) |

Banking'de bu standart claim'lerin yanına custom claim eklersin — tenant, session, MFA durumu gibi:

```json
{
  "tenant": "tr",
  "branch": "istanbul-1",
  "customer_id": "cust-456",
  "permissions": ["transfer.read", "transfer.write"],
  "session_id": "sess-789",
  "mfa_completed": true,
  "device_id": "device-abc"
}
```

Tuzak burada: payload base64url'dür, **şifreli değildir**. Kim eline geçirirse decode edip okur. <mark>JWT payload herkesin okuyabildiği açık bir zarftır — TC kimlik, kart PAN, bakiye gibi PII asla payload'a konmaz.</mark> PII gerekiyorsa token'a sadece `userId` koy, detayı API'den DB lookup ile çek.

### 3. JWS vs JWE

Neden önemli: "JWT" dendiğinde neredeyse her zaman JWS kastedilir; farkı bilmezsen "token imzalı, o zaman içerik gizli" yanılgısına düşersin.

**JWS (JSON Web Signature)** — yaygın olan. Token imzalıdır ama içerik okunabilir (base64url):

```
header.payload.signature
       ↑
    base64url decode = plain JSON
```

Tampering tespit edilir (signature mismatch), ama içerik gizli değildir.

**JWE (JSON Web Encryption)** — içerik şifrelidir; sadece yetkili taraf decrypt eder:

```
header.encrypted_key.iv.ciphertext.authentication_tag
```

Beş parça: AES-GCM gibi simetrik şifre + asymmetric key exchange kullanır.

Banking pratiği çoğunlukla **JWS + payload'da PII yok**'tur. PII gerekiyorsa JWE ile şişirmek yerine token'ı sadece bir referans (userId) tutup detayı API'den istemek daha ucuz ve daha az riskli.

### 4. Algoritmalar — HMAC vs RSA vs ECDSA

Neden önemli: algoritma seçimi "kim token üretebilir, kim doğrulayabilir" sorusunu belirler; yanlış seçim tüm sistemi bir secret sızıntısına açık hale getirir.

| Algorithm | Tür | Banking adoption |
|---|---|---|
| **HS256** | HMAC SHA-256, symmetric | Single-service (issuer = verifier, aynı secret) |
| HS384, HS512 | HMAC larger | Aynı, daha sıkı |
| **RS256** | RSA SHA-256, asymmetric | Microservice (banking standard) |
| RS384, RS512 | RSA larger | OK |
| ES256 | ECDSA P-256 SHA-256 | Daha kısa key, modern |
| PS256 | RSA-PSS | RSA varyasyonu |

**HS256 (symmetric):** issuer ve verifier aynı secret'ı paylaşır. Tek serviste basit ve hızlıdır, ama secret'ı doğrulaması gereken her servise dağıtman gerekir — bir servis sızarsa herkes geçerli token üretebilir.

**RS256 (asymmetric):** issuer **private key** ile imzalar, verifier'lar **public key** ile doğrular. Public key herkesle paylaşılabilir; private key sadece auth server'da kalır.

```
[Auth Server] - private key (gizli)
   ↓ token'ı private key ile imzalar
[Resource Server 1] - public key ile doğrular
[Resource Server 2] - public key ile doğrular
[Resource Server 3] - public key ile doğrular
```

<mark>Microservice mimaride imza için RS256 (asymmetric) kullan, paylaşılan secret gerektiren HS256'yı değil.</mark> Böylece doğrulayan servislerin hiçbiri token üretemez, sadece kontrol eder.

**Algorithm confusion attack:** Saldırgan RS256 token'ının header'ını `alg: HS256` yapar ve public key'i HMAC secret'ı gibi kullanarak imzalar. Naif verifier "alg HS256 demiş, HMAC ile doğrularım" der ve zaten herkese açık olan public key ile forge edilmiş token'ı kabul eder. Fix: verifier kabul edeceği algoritmayı sabitler.

```java
JwtParser parser = Jwts.parser()
    .verifyWith(publicKey)   // sadece bu key + onun algoritması
    .build();
// Spring Security OAuth2 Resource Server bunu doğru yapar
```

### 5. `alg: none` saldırısı

Neden önemli: JWT spec'i "imzasız token" için `alg: none` değerini tanımlar; naif bir verifier bunu görünce imza kontrolünü tamamen atlar.

```json
{"alg": "none", "typ": "JWT"}
```

Saldırgan istediği payload'ı yazar, signature kısmını boş bırakır, verifier "alg none, imza atla" der ve sahte token'ı kabul eder — tam bir authentication bypass.

```mermaid
flowchart LR
    A["Saldirgan token uretir"] --> B["Header alg none"]
    B --> C["Signature bos birakilir"]
    C --> D{"Verifier alg none kabul eder mi"}
    D -->|"Naive imza atla"| E["Sahte token gecerli sayilir"]
    D -->|"Guvenli whitelist"| F["Token reddedilir"]
```

Modern kütüphaneler bunu default'ta engeller, ama banking'de savunmayı garanti altına almak için manuel bir ban da eklenir:

```java
if ("none".equalsIgnoreCase(jwsHeader.getAlgorithm().getName())) {
    throw new SecurityException("Algorithm 'none' is not allowed");
}
```

```admonish warning title="alg=none ve algorithm confusion ortak dersi"
Her iki saldırı da aynı kök hatadan doğar: verifier'ın token header'ındaki `alg` değerine körü körüne güvenmesi. Kural sabit — doğrulanacak algoritmayı token değil, verifier belirler. Header'daki `alg`'a göre davranış seçen her kod potansiyel bypass'tır.
```

### 6. Token doğrulama akışı — Spring Security 6 Resource Server

Neden önemli: doğrulama tek adım değildir. İmza geçse bile süresi dolmuş, yanlış API'ye ait veya yanlış issuer'dan gelmiş bir token reddedilmelidir. <mark>Verifier kabul edeceği algoritmaları whitelist ile sabitler ve sırayla imza → exp → iss/aud kontrolü yapar; herhangi biri düşerse 401.</mark>

```mermaid
flowchart LR
    A["Gelen JWT"] --> B{"alg whitelist"}
    B -->|"Hayir"| R["401 reddet"]
    B -->|"Evet"| C{"Imza gecerli"}
    C -->|"Hayir"| R
    C -->|"Evet"| D{"exp taze"}
    D -->|"Hayir"| R
    D -->|"Evet"| E{"iss ve aud dogru"}
    E -->|"Hayir"| R
    E -->|"Evet"| S["Erisim ver"]
```

Spring Security 6'da Resource Server bu akışın hepsini yapar. Önce bağımlılık ve config:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

`issuer-uri` verildiğinde Spring `/.well-known/openid-configuration` üzerinden JWKS endpoint'ini keşfeder ve public key'leri otomatik çeker (rotation'a uyum sağlar):

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.mavibank.com/realms/banking
          jwk-set-uri: https://auth.mavibank.com/realms/banking/protocol/openid-connect/certs
```

Security filter chain'i stateless yapıp endpoint'leri scope/role'e bağlarsın:

```java
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
```

JWT içindeki claim'leri Spring authority'lerine çevirmek için bir converter yazarsın — `scope` claim'ini `SCOPE_` prefix'iyle, realm rollerini `ROLE_` ile map edersin:

```java
converter.setJwtGrantedAuthoritiesConverter(jwt -> {
    Set<GrantedAuthority> authorities = new HashSet<>();
    String scope = jwt.getClaimAsString("scope");
    if (scope != null) {
        for (String s : scope.split(" ")) {
            authorities.add(new SimpleGrantedAuthority("SCOPE_" + s));
        }
    }
    return authorities;
});
```

<details>
<summary>Tam kod: BankingResourceServerConfig + JwtAuthenticationConverter (~55 satır)</summary>

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
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            Set<GrantedAuthority> authorities = new HashSet<>();

            // Roles (realm_access.roles → ROLE_)
            Map<String, Object> realmAccess = jwt.getClaim("realm_access");
            if (realmAccess != null) {
                @SuppressWarnings("unchecked")
                List<String> roles = (List<String>) realmAccess.get("roles");
                if (roles != null) {
                    roles.forEach(r -> authorities.add(new SimpleGrantedAuthority("ROLE_" + r)));
                }
            }

            // Scopes (scope → SCOPE_)
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

</details>

Controller tarafında `@AuthenticationPrincipal Jwt` ile claim'lere doğrudan erişirsin:

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

Neden önemli: token'ı ne kadar uzun ömürlü yaparsan çalınması o kadar tehlikeli olur; ne kadar kısa yaparsan kullanıcıyı o kadar sık login'e zorlarsın. İki token'lı pattern bu ikilemi çözer.

**Access token** kısa ömürlüdür (5-15 dk) ve her API request'inde kullanılır. **Refresh token** uzun ömürlüdür (banking'de 1-24 saat, kısa tutulur) ve sadece yeni access token almak için kullanılır.

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Auth Server
    participant R as Resource Server
    C->>A: Login
    A-->>C: access 15dk ve refresh 1saat
    C->>R: API istegi access ile
    R-->>C: 200 OK
    Note over C: 16. dakika access expired
    C->>R: API istegi eski access
    R-->>C: 401 Unauthorized
    C->>A: POST auth refresh ile refresh token
    A-->>C: yeni access ve yeni refresh
    Note over A: Eski refresh USED isaretlenir
    C->>R: API istegi yeni access
    R-->>C: 200 OK
```

Banking pratiği net: access TTL 5-15 dk, refresh TTL 1-24 saat (sensitive olduğu için kısa), ve her refresh'te **yeni refresh token** üretilir. Bu son maddeye rotation denir ve bir sonraki konudur.

```admonish tip title="Refresh token'ı nerede saklamalı"
Refresh token access'ten daha değerlidir (yeni token üretir), o yüzden `localStorage`'a koyma — XSS ile çalınır. Web'de `HttpOnly`, `Secure`, `SameSite=Strict` cookie tercih et; JavaScript erişemez. Ayrıca refresh'i sadece `/auth/refresh` endpoint'inde kabul et, normal API request'lerinde asla — böylece kaza ile her yere taşınmaz.
```

### 8. Refresh token rotation + compromise detection

Neden önemli: rotation olmazsa çalınan bir refresh token, ömrü boyunca yeni access üretmeye devam eder ve bunu fark edemezsin. Rotation + reuse detection bu hırsızlığı görünür kılar.

**Rotation:** her refresh çağrısında eski refresh token "used" işaretlenir (one-time use) ve yeni bir çift verilir.

```
1. Login → A1 (access) + R1 (refresh)
2. A1 expired → POST /refresh (R1)
3. Server: R1'i doğrula → R1'i USED işaretle → A2 + R2 üret → döndür
4. A2 expired → POST /refresh (R2) → cycle devam
```

**Compromise detection** işin kalbidir: bir refresh token'ın ikinci kez kullanılması, ancak biri onu çalıp aynı token'ı yeniden denediği anlamına gelir. Çünkü meşru kullanıcı her zaman en son aldığı token'ı kullanır.

```
Saldırgan R1'i çalar.
Gerçek kullanıcı R1 kullanır → başarılı, R2 alır.
Saldırgan R1'i dener → SERVER: "R1 zaten kullanılmış! COMPROMISE."
  → kullanıcının TÜM refresh token'larını revoke et
  → re-login zorla + security ops'a alert
```

Kod tarafında reuse tespiti şöyle görünür — `record.isUsed()` true ise saldırı varsayılır:

```java
if (record.isUsed()) {
    // REUSE DETECTION → compromise
    log.error("Refresh token reuse detected: user={}, token={}", userId, tokenId);
    repo.revokeAllForUser(userId);
    alerts.notifySecurityOps("Possible refresh token theft: user=" + userId);
    notificationService.sendSecurityAlert(userId, "Suspicious activity, please re-login.");
    throw new SecurityException("Refresh token reuse detected");
}
```

Reuse değilse normal doğrulama + used işaretleme + yeni çift üretimi devam eder:

```java
if (record.isRevoked() || record.getExpiresAt().isBefore(Instant.now())) {
    throw new InvalidTokenException("Token revoked or expired");
}
record.setUsed(true);
record.setUsedAt(Instant.now());
repo.save(record);
return issueTokenPair(userId);
```

Banking standardı: refresh rotation + compromise detection **şart**, opsiyonel değil.

### 9. Token revocation — stateless'ın zayıf noktası

Neden önemli: stateless JWT'nin bedeli şudur — signature geçerli olduğu sürece token geçerlidir. Logout veya security incident'ta bir access token'ı nasıl anında iptal edersin? Üç strateji var.

**Yöntem 1 — Blocklist (Redis).** İptal edilen token'ın `jti`'sini, kalan ömrü kadar TTL ile Redis'e yazarsın:

```java
public void revoke(String jti, long expiresIn) {
    redis.opsForValue().set("blocked:" + jti, "1", Duration.ofSeconds(expiresIn));
    // TTL = token expiry → cleanup otomatik
}

public boolean isRevoked(String jti) {
    return redis.hasKey("blocked:" + jti);
}
```

Resource server her request'te blocklist kontrolü yapar:

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

Trade-off: her request bir Redis lookup (~1ms). Banking için kabul edilebilir.

**Yöntem 2 — Short-lived access + refresh revoke.** Access token 5 dakika olsun; compromise tespit edilince refresh'i anında revoke edersin, access de en geç 5 dakikada expire olur. Blocklist'e göre daha ucuzdur ama iptal anlık değildir.

**Yöntem 3 — Token versioning.** User'da bir `tokenVersion` alanı tut, her token'a gömü. Logout → version++. Eski version'lı token'lar invalid olur:

```java
// Issuance
JWT.create().withClaim("token_version", user.getTokenVersion()) /* ... */;

// Validation
if (jwt.getClaim("token_version").asInt() != user.getTokenVersion()) {
    throw new InvalidTokenException();
}
```

Basittir ama her request'te DB lookup gerektirir. Banking pratiği genelde **blocklist + short-lived combo**'dur.

### 10. Banking implementation — TokenService

Neden önemli: teoriyi RS256 ile üreten, rotate eden ve revoke eden tek bir servise oturttuğumuzda parçalar yerine oturur.

`init()` ile private/public key yüklenir, `issue()` access + refresh çiftini üretir. Access token'da business claim'ler (tenant, roles, scope) yer alır ve RS256 ile imzalanır:

```java
String accessToken = JWT.create()
    .withIssuer("https://auth.mavibank.com")
    .withSubject(user.getId().toString())
    .withAudience("banking-api")
    .withIssuedAt(now)
    .withExpiresAt(now.plus(15, ChronoUnit.MINUTES))
    .withJWTId(UUID.randomUUID().toString())
    .withClaim("tenant", user.getTenant())
    .withClaim("session_id", sessionId)
    .withArrayClaim("roles", user.getRoles().toArray(new String[0]))
    .withClaim("scope", String.join(" ", user.getScopes()))
    .sign(Algorithm.RSA256(publicKey, privateKey));
```

Refresh token ise minimaldir (`token_type: refresh`, uzun exp) ve DB'ye bir kayıt olarak saklanır — rotation ve reuse detection bu kayıt üzerinden yürür:

```java
refreshTokenRepo.save(RefreshTokenRecord.builder()
    .id(refreshTokenId)
    .userId(user.getId())
    .sessionId(UUID.fromString(sessionId))
    .issuedAt(now)
    .expiresAt(now.plus(1, ChronoUnit.HOURS))
    .used(false)
    .revoked(false)
    .build());
```

`refresh()` refresh token'ı RS256 ile doğrular, DB kaydını çeker, reuse/revoke kontrolünden geçirir ve yeni bir çift üretir. `revoke()` ise access token'ın `jti`'sini blocklist'e kalan ömrü kadar ekler.

<details>
<summary>Tam kod: TokenService (~90 satır)</summary>

```java
@Service
public class TokenService {

    @Value("${jwt.private-key-path}") private String privateKeyPath;
    @Value("${jwt.public-key-path}")  private String publicKeyPath;

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
            log.error("Refresh token reuse: user={}", userId);
            refreshTokenRepo.revokeAllForUser(userId);
            alerts.notifySecurityOps("Refresh token reuse: " + userId);
            throw new SecurityException("Token reuse detected");
        }
        if (record.isRevoked()) {
            throw new InvalidTokenException("Token revoked");
        }

        record.setUsed(true);
        record.setUsedAt(Instant.now());
        refreshTokenRepo.save(record);

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

</details>

### 11. JWKS — public key rotation

Neden önemli: auth server periyodik olarak imza anahtarını değiştirir (compromise durumunda anında); verifier'ların bu değişime elle müdahale olmadan uyum sağlaması gerekir.

**JWKS (JSON Web Key Set)** endpoint'i, geçerli public key'leri `kid` ile listeler:

```
GET /.well-known/jwks.json

{
  "keys": [
    {"kty": "RSA", "kid": "key-2025-q1", "alg": "RS256", "use": "sig", "n": "...", "e": "AQAB"},
    {"kty": "RSA", "kid": "key-2025-q2", "alg": "RS256", "use": "sig", "n": "...", "e": "AQAB"}
  ]
}
```

Token header'ındaki `kid` verifier'a hangi key ile doğrulayacağını söyler. Rotation sırasında eski ve yeni key bir süre birlikte yayınlanır ki geçişte token'lar kırılmasın. Spring Security `jwk-set-uri` ile bu seti otomatik çeker ve cache'ler (5 dk refresh). Banking pratiği: key rotation 3-12 ayda bir, compromise'de anında.

```admonish tip title="Rotation'da grace period"
Anahtarı değiştirdiğin an eski key ile imzalanmış token'lar hâlâ dolaşımdadır (access TTL kadar). JWKS'te eski ve yeni key'i, en az bir access token ömrü boyunca birlikte yayınla; yoksa geçiş anında geçerli kullanıcılar 401 yer. Verifier cache'i de bu süreyi hesaba katmalı — bilinmeyen bir `kid` görünce JWKS'i tazeleyip tekrar denemeli.
```

### 12. Banking security pitfalls

Neden önemli: JWT'nin çoğu güvenlik açığı kütüphane bug'ından değil, yanlış konfigürasyondan doğar. <mark>Her token `exp` taşımalı ve resource server `aud` claim'ini doğrulamalı — bu ikisi eksikse token sonsuza kadar ve her API'de geçerli olur.</mark>

**Pitfall 1 — `alg: none` (§5) ve Pitfall 2 — algorithm confusion (§4):** İkisi de explicit algorithm enforcement ile kapanır.

**Pitfall 3 — Weak HS256 secret.** `"secret"` gibi tahmin edilebilir bir key brute-force ile kırılır. HS256 minimum **32-byte (256-bit)** random secret ister:

```bash
openssl rand -base64 32
# → "Random32ByteString..."
```

**Pitfall 4 — `exp` yok.** `exp` claim'i olmayan token sonsuza kadar geçerlidir; çalınırsa hiç eskimez. Her token'da `exp` set olmalı.

**Pitfall 5 — Sensitive data payload'da.** Base64url decode = plaintext. TC kimlik, kart PAN, bakiye payload'da olursa herkes okur (§2). Banking'de payload'a sadece internal ID (UUID), role ve scope girer.

**Pitfall 6 — `aud` (audience) check yok.** `banking-api` için üretilmiş bir token, `shopping-api`'ye gönderilir; naif verifier imza geçerli diye kabul eder ve token yanlış serviste kullanılır. Fix: resource server `aud` claim'ini beklenen değere karşı kontrol eder:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.mavibank.com
          audiences: ["banking-api"]
```

**Pitfall 7 — Clock skew.** Auth server ile resource server saatleri kayıksa, henüz geçerli bir token "expired" ya da geçersiz bir token "not yet valid" görünebilir. Fix: cluster-wide NTP + verifier'da küçük bir leeway:

```java
JWT.require(/* ... */)
    .acceptLeeway(5)   // 5 sn tolerans
    .build();
```

```admonish warning title="Weak secret ve PII — en sık iki banking hatası"
Bu iki pitfall production incident'larının çoğunu oluşturur. HS256 secret'ı asla koda veya config dosyasına hardcode etme; vault'tan al ve en az 32-byte random olsun. Payload'a asla KVKK/PCI-DSS kapsamındaki veri (TC kimlik, kart PAN, bakiye) koyma; token bir referanstır, veri deposu değil.
```

### 13. JWE — ne zaman gerekir

Neden önemli: JWS'in "içerik okunabilir" özelliği çoğu zaman sorun değildir, ama bazı banking senaryolarında payload'ın gerçekten gizli kalması gerekir.

JWE banking'de nadirdir ama şu durumlarda değerlidir: highly sensitive payload (özel kart bilgisi), inter-bank communication, B2B Open Banking. Örneği:

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

Decryption private key ile yapılır. Trade-off: daha büyük token, daha yavaş işleme. Çoğu durumda **JWS + DB lookup** daha pragmatiktir.

### 14. Banking anti-pattern'leri

Mülakatta "bu JWT setup'ında ne yanlış?" sorusunun cephaneliği burasıdır. On klasik:

1. **HS256 microservice'lerde** — symmetric secret tüm servislere dağılır; bir servis sızarsa herkes token forge eder. Standart: RS256.
2. **HS256 hardcoded weak secret** — `jwt.secret: "secret"` tahmin edilebilir. Min 256-bit random, vault'tan.
3. **Sensitive data payload'da** — JWT base64url; PII = leak.
4. **Refresh rotation yok** — çalınan refresh sonsuza kadar access üretir.
5. **Compromise detection yok** — reuse sessizce yeni token üretir, saldırı görünmez.
6. **Token TTL çok uzun** — 24 saatlik access, çalınırsa 24 saat geçerli. Short access (5-15 dk) + refresh.
7. **`alg=none` accept** — modern lib'de kapalı ama config'le açılabilir; manuel ban ekle.
8. **`aud` check yok** — cross-service token reuse (§12 Pitfall 6).
9. **Logout sadece client-side** — `localStorage.removeItem('token')` token'ı server-side hâlâ geçerli bırakır; server-side revocation (blocklist) şart.
10. **Refresh token Authorization header'da** — `Authorization: Bearer <refresh>` refresh'i her request'e taşır. Refresh'i sadece `/auth/refresh` endpoint'ine, ayrı cookie/body ile kısıtla.

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

## Kendini Sına

Aşağıdaki soruları önce **cevaba bakmadan** kendi cümlelerinle yanıtlamayı dene — hepsi TR bank mülakatlarında karşına çıkabilecek tarzda. Takıldığın soru olursa ilgili Kavramlar başlığına dön, sonra tekrar dene.

**S1. Access token ile refresh token arasındaki fark nedir? Banking'de TTL'lerini nasıl seçersin ve neden iki ayrı token?**

<details>
<summary>Cevabı göster</summary>

Access token her API request'inde kullanılan, kısa ömürlü (banking'de 5-15 dk) token'dır. Refresh token ise sadece yeni access almak için kullanılan, daha uzun ömürlü (banking'de 1-24 saat, kısa tutulur) token'dır ve tek bir endpoint'te (`/auth/refresh`) çalışır.

İki token'ın sebebi bir ikilemi çözmektir: access'i kısa tutarsan çalınması az tehlikelidir (çabuk expire), ama kullanıcıyı sürekli login'e zorlarsın. Refresh bu boşluğu doldurur — access sık yenilenir ama kullanıcı yeniden şifre girmez. Banking sensitive olduğu için refresh de kısa tutulur ve her yenilemede rotate edilir.

</details>

**S2. `alg=none` saldırısı nedir, nasıl çalışır ve nasıl engellenir?**

<details>
<summary>Cevabı göster</summary>

JWT spec'i imzasız token için `alg: none` değerini tanımlar. Saldırgan istediği payload'ı yazar, header'a `alg: none` koyar ve signature kısmını boş bırakır. Naif bir verifier "algoritma none, imzayı atla" der ve sahte token'ı kabul eder — tam bir authentication bypass.

Fix: verifier token header'ındaki `alg`'a güvenmez, kabul edeceği algoritmaları kendisi sabitler (whitelist). Modern kütüphaneler bunu default'ta yapar; banking'de ayrıca `if alg == "none" throw` şeklinde manuel bir ban da eklenir. Kök ders: doğrulanacak algoritmayı token değil verifier belirlemelidir.

</details>

**S3. Algorithm confusion attack nedir? `alg=none` ile ortak noktası ne?**

<details>
<summary>Cevabı göster</summary>

RS256 kullanan bir sistemde saldırgan token header'ını `alg: HS256` yapar ve zaten herkese açık olan public key'i HMAC secret'ı gibi kullanarak token'ı imzalar. Naif verifier "HS256 demiş, HMAC ile doğrularım" der ve public key ile forge edilmiş token'ı geçerli sayar. Yani asymmetric sistem, saldırganın elindeki public bilgiyle sahtelenmiş olur.

`alg=none` ile ortak kök hata aynı: verifier'ın token'daki `alg` değerine körü körüne güvenmesi. İkisi de "verifier kabul edeceği algoritmayı sabitler" (explicit algorithm enforcement) ile kapanır. Spring Security OAuth2 Resource Server key'e bağlı doğrulama yaptığı için bunu doğru yapar.

</details>

**S4. HS256 (HMAC) ile RS256 (RSA) arasında ne fark var? Microservice mimaride hangisini mutlaka seçersin?**

<details>
<summary>Cevabı göster</summary>

HS256 symmetric'tir: imzalayan ve doğrulayan aynı secret'ı paylaşır. RS256 asymmetric'tir: auth server private key ile imzalar, doğrulayanlar public key ile kontrol eder — public key herkesle paylaşılabilir, private key sadece issuer'da.

Microservice'te mutlaka RS256. Çünkü HS256'da secret'ı doğrulaması gereken her servise dağıtman gerekir; bu servislerden biri compromise olursa saldırgan geçerli token üretebilir. RS256'da doğrulayan servislerin elinde sadece public key vardır — token'ı kontrol edebilir ama üretemezler. HS256 sadece issuer = verifier olan tek serviste makuldür.

</details>

**S5. Stateless JWT'de bir token'ı logout'ta veya security incident'ta nasıl iptal edersin? Hangi stratejiler var?**

<details>
<summary>Cevabı göster</summary>

Stateless JWT'nin zayıf noktası: signature geçerli olduğu sürece token geçerlidir, o yüzden iptal ekstra mekanizma ister. Üç strateji: (1) Redis blocklist — iptal edilen token'ın `jti`'sini kalan ömrü kadar TTL ile Redis'e yazarsın, resource server her request'te kontrol eder (~1ms, cleanup TTL ile otomatik). (2) Short-lived access + refresh revoke — access 5 dk olsun, incident'ta refresh'i revoke et, access en geç 5 dk'da expire olur. (3) Token versioning — user'da `tokenVersion` alanı, logout'ta ++; eski version'lı token'lar invalid ama her request DB lookup ister.

Banking pratiği genelde blocklist + short-lived combo'dur: anlık iptal (blocklist) ile ucuz güvenlik ağı (kısa TTL) bir arada.

</details>

**S6. Refresh token rotation nedir ve compromise detection nasıl çalışır?**

<details>
<summary>Cevabı göster</summary>

Rotation: her refresh çağrısında eski refresh token "used" olarak işaretlenir (one-time use) ve yeni bir access + refresh çifti verilir. Böylece bir refresh token yalnızca bir kez yeni token üretebilir.

Compromise detection bunun üzerine kurulur: bir refresh token ikinci kez kullanılmaya çalışılırsa, bu ancak biri onu çaldığı anlamına gelir — çünkü meşru kullanıcı her zaman en son aldığı token'ı kullanır. Server `record.isUsed()` true görürse saldırı varsayar: kullanıcının tüm refresh token'larını revoke eder, re-login zorlar ve security ops'a alert atar. Rotation olmadan çalınan refresh sessizce sonsuza kadar access üretir; detection olmadan da bu hırsızlık hiç görünmez. Banking'de ikisi de şart.

</details>

**S7. `aud` (audience) claim'i neden önemli? Eksikse hangi saldırı mümkün olur?**

<details>
<summary>Cevabı göster</summary>

`aud` claim'i token'ın hangi API için üretildiğini söyler (örn. `banking-api`). Resource server bu değeri kendi beklediği audience'a karşı kontrol etmezse cross-service token reuse mümkün olur: `banking-api` için üretilmiş bir token `shopping-api`'ye gönderilir, imza geçerli olduğu için naif verifier kabul eder ve token yetkisiz bir serviste kullanılır.

Fix: her resource server yalnızca kendi audience'ına ait token'ları kabul eder (`audiences: ["banking-api"]`). Aynı auth server birden fazla API'ye token üretiyorsa bu kontrol zorunludur, aksi halde bir servisin token'ı diğerinin kapısını açar.

</details>

**S8. JWT payload'a neyi koyabilir, neyi asla koyamazsın? Neden?**

<details>
<summary>Cevabı göster</summary>

Payload base64url'dür, şifreli değildir — token'ı eline geçiren herkes decode edip okur (JWS). Bu yüzden PII veya sensitive data asla payload'a konmaz: TC kimlik, kart PAN, bakiye, adres gibi KVKK/PCI-DSS kapsamındaki her şey yasaktır. Konursa base64url decode = düz metin sızıntısı.

Payload'a sadece internal referanslar girer: user ID (UUID), roller, scope'lar ve tenant/session gibi kimlik context'i. Detay veri gerekiyorsa token sadece `userId` taşır, gerçek veriyi API DB'den lookup ile çeker. Gerçekten şifreli payload şartsa (nadir: inter-bank, B2B Open Banking) JWS yerine JWE kullanılır ama bu daha büyük ve yavaş token demektir.

</details>

---

## Tamamlama kriterleri

- [ ] "Kendini Sına" bölümündeki tüm soruları cevaba bakmadan açıklayabiliyorum
- [ ] JWT'nin üç parçasını ve payload'ın neden "imzalı ama okunabilir" olduğunu anlatabiliyorum
- [ ] HS256 vs RS256 farkını ve microservice'te neden RS256 seçtiğimi söyleyebiliyorum
- [ ] `alg=none` ve algorithm confusion saldırılarını + ortak fix'lerini açıklayabiliyorum
- [ ] Access + refresh pattern, refresh rotation ve compromise detection'ı tahtada çizebilirim
- [ ] Üç revocation stratejisini (blocklist, short-lived, versioning) trade-off'larıyla sayabiliyorum
- [ ] 7 pitfall'ı (weak secret, PII, eksik exp/aud, clock skew, alg confusion, alg=none) tanıyabiliyorum
- [ ] JWKS + `kid` ile key rotation'ın verifier tarafında nasıl otomatik çalıştığını biliyorum
- [ ] (Opsiyonel) "Pratik yapmak istersen" bölümündeki testleri yazdım ve Claude-verify prompt'uyla doğrulattım

---

## Defter notları

1. "JWT 3 parça (header.payload.signature) — payload neden şifreli değil: ____."
2. "JWS vs JWE ne zaman hangisi: ____."
3. "HS256 vs RS256 microservice senaryosu: ____."
4. "Algorithm confusion attack + explicit enforcement fix: ____."
5. "`alg=none` saldırısı nasıl bypass eder, nasıl engellenir: ____."
6. "Access + refresh pattern banking TTL kararı: ____."
7. "Refresh rotation + compromise detection mekanizması: ____."
8. "Token revocation 3 strateji (blocklist / short-lived / versioning): ____."
9. "`aud` claim cross-service token reuse prevention: ____."
10. "JWKS endpoint + `kid` ile key rotation, verifier auto-fetch: ____."

```admonish success title="Bölüm Özeti"
- JWT üç base64url parçadır (header.payload.signature); payload imzalıdır ama **şifreli değildir** — asla PII koyma, token bir referanstır
- Microservice'te RS256 (asymmetric) standarttır: private key imzalar, public key doğrular; HS256'nın paylaşılan secret'ı tek servis dışında risklidir
- `alg=none` ve algorithm confusion aynı kök hatadan doğar — verifier `alg`'a güvenmemeli, kabul edeceği algoritmayı sabitlemeli
- Access (5-15 dk) + refresh (1-24 saat) pattern + rotation + reuse ile compromise detection banking'de şarttır
- Stateless token'ı geri almanın yolu revocation: Redis blocklist + short-lived combo en yaygın banking çözümüdür
- Pitfall cephaneliği: weak secret, PII payload, eksik `exp`/`aud`, clock skew, client-side-only logout — hepsi config hatasıdır, kütüphane bug'ı değil
```

---

## Pratik yapmak istersen

Kavramları koda dökmek istersen aşağıdaki iki ek hazır: test yazma rehberi RS256 issuance, refresh rotation, compromise detection, expired/wrong-audience reddi ve PII-yok kontrolü için örnek testler içerir; Claude-verify prompt'u ile yazdığın JWT kodunu banking-grade perspektiften denetletebilirsin. Önce RSA key pair üretmen gerekir:

```bash
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

> Production'da key'leri koda veya repoya yazma; vault'tan (Topic 8.x) al. Bu dosyalar sadece lokal deneme içindir.

<details>
<summary>Test yazma rehberi</summary>

### Test 8.3.1 — RS256 token pair issuance

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
}
```

### Test 8.3.2 — Refresh rotation + DB tracking

```java
@Test
void shouldRotateRefreshToken() {
    TokenPair pair1 = tokenService.issue(createTestUser());

    TokenPair pair2 = tokenService.refresh(pair1.refreshToken());

    assertThat(pair2.refreshToken()).isNotEqualTo(pair1.refreshToken());

    // Eski refresh artık USED
    UUID oldTokenId = UUID.fromString(JWT.decode(pair1.refreshToken()).getId());
    RefreshTokenRecord oldRecord = repo.findById(oldTokenId).orElseThrow();
    assertThat(oldRecord.isUsed()).isTrue();
}
```

### Test 8.3.3 — Compromise detection (reuse → revoke all)

```java
@Test
void shouldDetectRefreshTokenReuse() {
    User user = createTestUser();
    TokenPair pair1 = tokenService.issue(user);
    TokenPair pair2 = tokenService.refresh(pair1.refreshToken());

    // pair1'in refresh'ini tekrar dene → compromise
    assertThatThrownBy(() -> tokenService.refresh(pair1.refreshToken()))
        .isInstanceOf(SecurityException.class);

    // Kullanıcının tüm refresh token'ları revoke edilmiş olmalı
    UUID pair2TokenId = UUID.fromString(JWT.decode(pair2.refreshToken()).getId());
    RefreshTokenRecord pair2Record = repo.findById(pair2TokenId).orElseThrow();
    assertThat(pair2Record.isRevoked()).isTrue();
}
```

### Test 8.3.4 — Expired ve wrong-audience token reddi

Resource server'ın süresi dolmuş ve yanlış audience'lı token'ları reddettiğini `mockMvc` ile doğrula:

```java
@Test
void shouldRejectExpiredAccessToken() throws Exception {
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
```

### Test 8.3.5 — PII payload'da YOK kontrolü

```java
@Test
void shouldNotIncludeSensitiveDataInPayload() {
    User user = User.builder()
        .id(UUID.randomUUID())
        .tcKimlik("12345678901")
        .cardPan("4111-1111-1111-1234")
        .build();

    TokenPair pair = tokenService.issue(user);

    DecodedJWT decoded = JWT.decode(pair.accessToken());
    String claims = decoded.getClaims().toString();
    assertThat(claims).doesNotContain("12345678901");
    assertThat(claims).doesNotContain("4111-1111-1111");
}
```

### Bonus — Token blocklist ve algorithm confusion

- **Blocklist:** logout sonrası token'ın `jti`'sini Redis'e ekle, aynı token'la request'in 401 döndüğünü doğrula (TTL = token expiry).
- **Algorithm confusion:** RS256 public key'i HS256 secret'ı gibi kullanarak forge edilmiş bir token üret; resource server'ın bunu reddettiğini doğrula.

</details>

<details>
<summary>Claude-verify prompt</summary>

```
Aşağıdaki JWT implementation'ımı banking-grade kriterlere göre değerlendir.
Eksikleri işaretle, kod yazma:

1. Algorithm:
   - RS256 (asymmetric) microservice'lerde mi?
   - HS256 hardcoded weak secret YOK mu?
   - Explicit algorithm enforcement (alg confusion prevention) var mı?
   - alg=none accept edilmiyor mu?

2. Standard claims:
   - exp claim her token'da mı?
   - iss, aud, sub explicit mi?
   - jti unique (revocation için) mi?
   - Clock skew leeway (5-30 sn) var mı?

3. Payload:
   - PII (TC No, card PAN, balance) payload'da YOK mu?
   - Sadece internal ID (UUID), role, scope mu?

4. Access + Refresh pattern:
   - Access TTL 5-15 dk mı?
   - Refresh TTL 1-24 saat (banking için kısa) mı?
   - Refresh rotation (her refresh'te yeni R) var mı?

5. Compromise detection:
   - Used refresh token reuse → SecurityException mı?
   - All user tokens revoke ediliyor mu?
   - Security ops alert var mı?

6. Token revocation:
   - Blocklist (Redis) ile logout mu?
   - TTL token expiry'siyle aynı (cleanup) mı?
   - Resource server filter blocklist check yapıyor mu?

7. Spring Security:
   - oauth2ResourceServer JWT setup var mı?
   - JwtAuthenticationConverter realm_access + scope mapping mi?
   - Stateless session mı?

8. JWKS:
   - Auth server JWKS endpoint var mı?
   - Resource server jwk-set-uri auto-fetch mi?
   - Key rotation stratejisi var mı?

9. Banking-specific:
   - Refresh token sadece /auth/refresh endpoint'inde mi kullanılıyor?
   - Audience check production'da var mı?
   - Custom claim (tenant, session_id) banking için mi?

10. Anti-pattern:
    - HS256 microservice'te mi?
    - Weak secret var mı?
    - Long access TTL (24+ saat) var mı?
    - Refresh rotation yok mu?
    - Sensitive payload var mı?
    - alg=none accept ediliyor mu?
    - aud check yok mu?
    - Client-side only logout mu?

Her madde için PASS / FAIL / EKSIK işaretle, kanıt göster, kod yazma.
```

</details>
