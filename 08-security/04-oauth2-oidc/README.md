# Topic 8.4 — OAuth2 & OIDC: Authorization Framework

## Hedef

OAuth 2.0 / OAuth 2.1 authorization framework ve OpenID Connect (OIDC) authentication layer'ı banking-grade derinlikte öğrenmek. Authorization Code + PKCE, Client Credentials, refresh flow, scopes, banking için Open Banking (BDDK 2020+) ve FAPI (Financial-grade API) standartları.

## Süre

Okuma: 2 saat • Mini task: 2 saat • Test: 1 saat • Toplam: ~5 saat

## Önbilgi

- Topic 8.3 (JWT) bitti
- Topic 8.2 (Authentication) bitti
- HTTP redirect, query parameter, URL encoding temel

---

## Kavramlar

### 1. OAuth 2.0 — ne olmadığını anlayalım

OAuth 2.0 **authorization framework** — kullanıcı yerine başka uygulamanın limited access alması.

OAuth **authentication değil** — "kim olduğun"u kanıtlamak için OIDC katmanı eklenir.

**Banking örnek:**
```
Kullanıcı: Maven Bank müşterisi
Third-party app: "Hesap Yöneticisi" (mint.com benzeri)
İstek: "Maven Bank hesap verisi okuma" izni
```

Naive yaklaşım:
```
Üçüncü uygulama Maven Bank şifresini sakla → KÖTÜ
- Şifre değişimi → uygulama kırılır
- Uygulama compromise → şifre leak
- "Hesap oku" izni vermek istedi, "hesap aktar" da yapabilir
```

OAuth 2.0 çözümü:
```
Maven Bank "access token" verir →
- Limited scope: sadece "account.read"
- Time-bound: 15 dk
- Revoke edilebilir
- Şifre asla third-party'ye gitmez
```

### 2. OAuth 2.0 roles

| Role | Banking örnek |
|---|---|
| **Resource Owner** | User (banking customer) |
| **Resource Server** | Banking API (`api.mavibank.com`) |
| **Client** | Third-party app (hesap yöneticisi mobile app) |
| **Authorization Server** | Banking auth (`auth.mavibank.com`) |

### 3. OAuth 2.0 grant types (flows)

| Grant Type | Banking Use Case | Status |
|---|---|---|
| **Authorization Code** | User flow (third-party web/mobile) | Standard |
| **Authorization Code + PKCE** | SPA, mobile, public client | OAuth 2.1 mandatory |
| **Client Credentials** | Service-to-service | Banking internal |
| **Refresh Token** | Token rotation | Standard |
| **Resource Owner Password** | User direct credentials | DEPRECATED |
| **Implicit** | SPA legacy | DEPRECATED (OAuth 2.1) |
| **Device Code** | TV, IoT, kiosk | Banking nadir |

### 4. Authorization Code flow — detaylı

```
Step 1: User clicks "Login with Maven Bank" in third-party app
  
Step 2: Third-party redirects to auth server
  GET https://auth.mavibank.com/oauth2/authorize
    ?response_type=code
    &client_id=hesap-yoneticisi
    &redirect_uri=https://hesapyoneticisi.com/callback
    &scope=account.read transactions.read
    &state=random_csrf_token
    &code_challenge=BASE64URL(SHA256(code_verifier))
    &code_challenge_method=S256

Step 3: User authenticates on auth.mavibank.com (login + password + MFA)

Step 4: Auth server shows consent screen
  "Hesap Yöneticisi wants to access:
    - Account information (read-only)
    - Transaction history (read-only)
   [Approve] [Deny]"
   
Step 5: User approves. Auth server redirects back with code:
  HTTP 302
  Location: https://hesapyoneticisi.com/callback
    ?code=AUTH_CODE_xyz   (one-time use, 30-60 sn TTL)
    &state=random_csrf_token

Step 6: Third-party server exchanges code for tokens (BACK-CHANNEL)
  POST https://auth.mavibank.com/oauth2/token
    Authorization: Basic base64(client_id:client_secret)
    Content-Type: application/x-www-form-urlencoded
    grant_type=authorization_code
    code=AUTH_CODE_xyz
    redirect_uri=https://hesapyoneticisi.com/callback
    code_verifier=ORIGINAL_VERIFIER
  
  Response:
  {
    "access_token": "eyJ...",
    "refresh_token": "eyJ...",
    "token_type": "Bearer",
    "expires_in": 900,
    "scope": "account.read transactions.read",
    "id_token": "eyJ..."
  }

Step 7: Third-party uses access_token for API calls
  GET https://api.mavibank.com/v1/accounts
    Authorization: Bearer eyJ...
```

### 5. PKCE — Proof Key for Code Exchange

**Problem:** Authorization Code public client (mobile/SPA)'da intercepted edilebilir. Saldırgan code'u alıp tokenlere swap eder.

**PKCE çözüm:**

```
Step A (client başında):
  code_verifier = random 43-128 char
  code_challenge = BASE64URL(SHA256(code_verifier))

Step B (authorize request):
  Send: code_challenge

Step C (token request):
  Send: code_verifier
  
Auth server: SHA256(code_verifier) === code_challenge?
  Match → issue tokens
  No match → reject
```

Saldırgan code'u çalsa bile, `code_verifier` ona yok → token alamaz.

**OAuth 2.1:** PKCE **mandatory** (her authorization code flow için).

Banking için: mobile + SPA = **PKCE şart**.

### 6. Client Credentials flow — service-to-service

User involvement yok. Server-to-server.

```
POST /oauth2/token
Authorization: Basic base64(client_id:client_secret)
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
scope=internal.transfer.write

Response:
{
  "access_token": "eyJ...",
  "expires_in": 300,
  "scope": "internal.transfer.write"
}
```

Token'da `sub` user değil, **client_id**.

Banking örnek:
- `payment-service` `account-service`'i çağırıyor → client credentials
- Internal scheduler external API → client credentials

**mTLS varyasyon:** Banking sıkı setup'larda client credentials **mTLS** ile birlikte (certificate-bound token).

### 7. OIDC — OpenID Connect

OAuth 2.0 + identity layer.

**ID Token** (JWT) — kullanıcı kimlik bilgisi.

```json
{
  "iss": "https://auth.mavibank.com",
  "sub": "user-123",
  "aud": "client-id",
  "exp": 1716003600,
  "iat": 1716000000,
  "auth_time": 1715999999,
  "nonce": "random",
  "name": "Ahmet Yılmaz",
  "email": "ahmet@example.com",
  "email_verified": true,
  "preferred_username": "ahmet_y"
}
```

**OAuth (sadece):** "Bu client'a X scope verildi."
**OIDC:** "Kullanıcı kim, ne zaman authenticate oldu."

#### OIDC scopes

- `openid` — OIDC enable (mandatory)
- `profile` — name, picture, etc.
- `email` — email + email_verified
- `phone` — phone_number
- `address` — addresses

#### Authorization endpoint

```
GET /oauth2/authorize
  ?response_type=code
  &client_id=...
  &redirect_uri=...
  &scope=openid profile email account.read
  &state=...
  &nonce=...
  &code_challenge=...
  &code_challenge_method=S256
```

#### UserInfo endpoint

```
GET https://auth.mavibank.com/userinfo
Authorization: Bearer <access_token>

Response:
{
  "sub": "user-123",
  "name": "Ahmet Yılmaz",
  "email": "ahmet@example.com",
  ...
}
```

ID token'da olmayan ek claim'leri buradan al.

### 8. Discovery — `/.well-known/openid-configuration`

Standart endpoint. Tüm config burada:

```
GET https://auth.mavibank.com/.well-known/openid-configuration

{
  "issuer": "https://auth.mavibank.com",
  "authorization_endpoint": "https://auth.mavibank.com/oauth2/authorize",
  "token_endpoint": "https://auth.mavibank.com/oauth2/token",
  "userinfo_endpoint": "https://auth.mavibank.com/userinfo",
  "jwks_uri": "https://auth.mavibank.com/.well-known/jwks.json",
  "revocation_endpoint": "https://auth.mavibank.com/oauth2/revoke",
  "introspection_endpoint": "https://auth.mavibank.com/oauth2/introspect",
  "end_session_endpoint": "https://auth.mavibank.com/logout",
  "response_types_supported": ["code", "id_token", "code id_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256", "ES256"],
  "scopes_supported": ["openid", "profile", "email", "account.read"],
  "grant_types_supported": ["authorization_code", "client_credentials", "refresh_token"],
  "code_challenge_methods_supported": ["S256"]
}
```

Spring `issuer-uri` config ile bu otomatik fetch.

### 9. Scopes — banking design

Banking için fine-grained scope:

```
account.read           — Hesap bilgisi oku
account.balance.read   — Sadece balance
transactions.read      — Transaction history oku
transactions.read.recent  — Sadece son 30 gün
transfer.write         — Para transferi yap
transfer.write.eft     — Sadece EFT
card.read              — Kart bilgisi
card.write             — Kart işlemi (blok/açma)
profile.read           — Kullanıcı profili
profile.write          — Profil güncelle (limited)
```

**Banking pratiği:**
- En küçük gerekli scope iste (least privilege)
- Sensitive scope (transfer.write) → ek MFA challenge
- Open Banking standardı (BDDK) için standart scope'lar

### 10. Spring Authorization Server (banking auth server)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-authorization-server</artifactId>
</dependency>
```

```java
@Configuration
@EnableWebSecurity
public class AuthServerConfig {
    
    @Bean
    @Order(1)
    public SecurityFilterChain authServerSecurityFilterChain(HttpSecurity http) throws Exception {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);
        http.getConfigurer(OAuth2AuthorizationServerConfigurer.class)
            .oidc(Customizer.withDefaults());
        http.exceptionHandling(e -> 
            e.authenticationEntryPoint(new LoginUrlAuthenticationEntryPoint("/login")));
        return http.build();
    }
    
    @Bean
    @Order(2)
    public SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(a -> a
                .requestMatchers("/login", "/error", "/.well-known/**").permitAll()
                .anyRequest().authenticated())
            .formLogin(Customizer.withDefaults());
        return http.build();
    }
    
    @Bean
    public RegisteredClientRepository registeredClientRepository() {
        RegisteredClient hesapYoneticisi = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("hesap-yoneticisi")
            .clientSecret("{bcrypt}$2a$10$...")
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .redirectUri("https://hesapyoneticisi.com/login/oauth2/code/mavibank")
            .scope(OidcScopes.OPENID)
            .scope(OidcScopes.PROFILE)
            .scope("account.read")
            .scope("transactions.read")
            .clientSettings(ClientSettings.builder()
                .requireAuthorizationConsent(true)
                .requireProofKey(true)
                .build())
            .tokenSettings(TokenSettings.builder()
                .accessTokenTimeToLive(Duration.ofMinutes(15))
                .refreshTokenTimeToLive(Duration.ofHours(1))
                .reuseRefreshTokens(false)
                .build())
            .build();
        
        RegisteredClient internalPayment = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("payment-service")
            .clientSecret("{bcrypt}...")
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
            .scope("internal.account.read")
            .scope("internal.transfer.write")
            .tokenSettings(TokenSettings.builder()
                .accessTokenTimeToLive(Duration.ofMinutes(5))
                .build())
            .build();
        
        return new InMemoryRegisteredClientRepository(hesapYoneticisi, internalPayment);
    }
    
    @Bean
    public JWKSource<SecurityContext> jwkSource() {
        RSAKey rsaKey = generateRsa();
        JWKSet jwkSet = new JWKSet(rsaKey);
        return new ImmutableJWKSet<>(jwkSet);
    }
    
    @Bean
    public AuthorizationServerSettings authorizationServerSettings() {
        return AuthorizationServerSettings.builder()
            .issuer("https://auth.mavibank.com")
            .build();
    }
    
    @Bean
    public OAuth2TokenCustomizer<JwtEncodingContext> jwtTokenCustomizer() {
        return context -> {
            if (context.getTokenType().getValue().equals(OAuth2TokenType.ACCESS_TOKEN.getValue())) {
                JwtClaimsSet.Builder claims = context.getClaims();
                Authentication principal = context.getPrincipal();
                
                if (principal.getPrincipal() instanceof User user) {
                    claims.claim("tenant", user.getTenant());
                    claims.claim("customer_id", user.getCustomerId());
                    claims.claim("mfa_completed", user.isMfaCompleted());
                }
            }
        };
    }
}
```

### 11. Spring Security OAuth2 Client (third-party app)

Üçüncü taraf app banking auth ile login:

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          mavibank:
            client-id: hesap-yoneticisi
            client-secret: ${BANK_CLIENT_SECRET}
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/mavibank"
            scope: openid,profile,account.read,transactions.read
        provider:
          mavibank:
            issuer-uri: https://auth.mavibank.com
```

```java
@Configuration
@EnableWebSecurity
public class ClientWebSecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(a -> a
                .requestMatchers("/", "/login").permitAll()
                .anyRequest().authenticated())
            .oauth2Login(o -> o
                .userInfoEndpoint(u -> u.oidcUserService(oidcUserService())))
            .oauth2Client(Customizer.withDefaults());
        return http.build();
    }
    
    @Bean
    public OidcUserService oidcUserService() {
        return new OidcUserService();
    }
}
```

Controller — banking API call:

```java
@RestController
public class AccountController {
    
    private final WebClient webClient;
    
    public AccountController(OAuth2AuthorizedClientManager authorizedClientManager) {
        ServletOAuth2AuthorizedClientExchangeFilterFunction oauth2Filter = 
            new ServletOAuth2AuthorizedClientExchangeFilterFunction(authorizedClientManager);
        oauth2Filter.setDefaultClientRegistrationId("mavibank");
        
        this.webClient = WebClient.builder()
            .filter(oauth2Filter)
            .baseUrl("https://api.mavibank.com")
            .build();
    }
    
    @GetMapping("/my-accounts")
    public Flux<Account> getAccounts(@RegisteredOAuth2AuthorizedClient("mavibank") 
                                     OAuth2AuthorizedClient client) {
        return webClient.get()
            .uri("/v1/accounts")
            .retrieve()
            .bodyToFlux(Account.class);
    }
}
```

### 12. FAPI — Financial-grade API

Open Banking için sıkı standart. Standard OAuth + ek güvenlik:

**FAPI Baseline:**
- HTTPS mandatory
- Bearer token Authorization header
- Audience validation strict

**FAPI Advanced:**
- mTLS client authentication
- JWT-secured authorization request (JAR) — request signed
- Signed JWT response (JARM)
- PKCE mandatory
- Certificate-bound access token

```
Token issued for Client A's certificate.
Resource server checks: does request come with Client A's mTLS cert?
  Match → accept
  No match → reject (certificate-bound)
```

Banking TR Open Banking BDDK düzenlemeleri 2020 sonrası bunu adopting.

### 13. Token introspection

Opaque token (JWT olmayan) için validation:

```
POST /oauth2/introspect
Authorization: Basic base64(client_id:client_secret)
Content-Type: application/x-www-form-urlencoded

token=opaque_token_xyz

Response:
{
  "active": true,
  "sub": "user-123",
  "scope": "account.read",
  "exp": 1716003600,
  "client_id": "hesap-yoneticisi",
  "username": "ahmet"
}
```

**JWT vs Introspection:**

| | JWT | Introspection |
|---|---|---|
| Validation | Local (signature) | Remote call |
| Revocation | Blocklist gerekli | Anlık |
| Latency | Sıfır | ~ms |
| Network | Independent | Auth server'a bağlı |
| Banking | Yaygın | Bazı durumlar |

Banking için: **JWT + blocklist** çoğu zaman; **introspection** yüksek-revocation-need senaryolarda.

### 14. Logout — OIDC RP-Initiated Logout

```
GET https://auth.mavibank.com/connect/logout
  ?id_token_hint=eyJ...
  &post_logout_redirect_uri=https://hesapyoneticisi.com/

Auth server:
  - Invalidate user session
  - Notify Relying Parties (back-channel logout)
  - Redirect to post_logout_redirect_uri
```

**Back-channel logout:**

Auth server, kullanıcı oturum açtığı tüm RP'lere logout token gönderir:

```
POST https://hesapyoneticisi.com/logout/back-channel
Content-Type: application/x-www-form-urlencoded

logout_token=eyJ...
```

Banking için: SSO logout cluster-wide önemli.

### 15. Banking — OAuth/OIDC anti-pattern'leri

**Anti-pattern 1: Resource Owner Password grant**

```
POST /token
grant_type=password
username=user
password=secret
```

Third-party şifre öğrenir — OAuth'un karşı çıktığı şey. **OAuth 2.1 deprecated.**

**Anti-pattern 2: Implicit grant**

```
?response_type=token   (access_token URL fragment'inde direkt döner)
```

URL'de token → browser history, log leak. **OAuth 2.1 deprecated.** Auth Code + PKCE kullan.

**Anti-pattern 3: PKCE'siz public client**

Mobile/SPA için PKCE şart.

**Anti-pattern 4: `state` parameter yok**

CSRF açığı. State random + session'la verify.

**Anti-pattern 5: Wildcard redirect_uri**

```
redirect_uri=https://*.example.com/cb
```

Subdomain takeover → token theft. **Exact match.**

**Anti-pattern 6: Client secret mobile app'te**

Mobile public client → secret reverse-engineer edilebilir. **PKCE only.**

**Anti-pattern 7: Long access token TTL**

Banking: 5-15 dk max.

**Anti-pattern 8: Refresh token rotation yok**

Topic 8.3'te.

**Anti-pattern 9: Scope too broad**

`account.write` (her şey) yerine `transfer.write`, `card.write` gibi fine-grained.

**Anti-pattern 10: Audience check yok**

Resource server `aud` validation şart.

---

## Önemli olabilecek araştırma kaynakları

- RFC 6749 — OAuth 2.0
- RFC 7636 — PKCE
- RFC 8252 — OAuth for Native Apps
- OAuth 2.1 draft
- OpenID Connect Core 1.0
- FAPI specification
- BDDK Open Banking
- Spring Authorization Server docs

---

## Mini task'ler

### Task 8.4.1 — Spring Authorization Server kurulumu (60 dk)

Spring Authorization Server starter ekle. AuthServerConfig yaz. `/.well-known/openid-configuration` reach.

### Task 8.4.2 — RegisteredClient (60 dk)

- `hesap-yoneticisi` (Auth Code + PKCE + Refresh + scope account.read)
- `payment-service` (Client Credentials + scope internal.transfer.write)

### Task 8.4.3 — Authorization Code + PKCE flow (60 dk)

Postman / curl ile end-to-end test:
1. Generate code_verifier + code_challenge
2. GET /authorize → login + consent
3. POST /token with code + verifier → tokens
4. Decode access + id token claims

### Task 8.4.4 — Spring Security OAuth2 Client (45 dk)

Üçüncü taraf app simulate. `oauth2Login()` ile login flow. Login sonrası `/my-accounts` resource server'dan veri çek.

### Task 8.4.5 — Client Credentials flow (30 dk)

`payment-service` → `account-service` arası. Internal service-to-service token issuance + scope check.

### Task 8.4.6 — Consent screen + audit (45 dk)

Custom consent UI. User'ın hangi scope'a onay verdiği audit table'a kaydedilsin (banking compliance).

### Task 8.4.7 — Token introspection endpoint (30 dk)

`POST /oauth2/introspect` opaque token validation. Resource server JWT yerine introspection kullansın.

### Task 8.4.8 — Logout flow (30 dk)

OIDC RP-initiated logout. Session invalidate. Back-channel logout endpoint.

---

## Test yazma rehberi

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
class OAuth2FlowTest {
    
    @LocalServerPort int port;
    
    @Test
    void authorizationCodeFlowWithPkce() throws Exception {
        String codeVerifier = generateCodeVerifier();
        String codeChallenge = sha256Base64Url(codeVerifier);
        
        String authUrl = "http://localhost:" + port + "/oauth2/authorize"
            + "?response_type=code"
            + "&client_id=hesap-yoneticisi"
            + "&redirect_uri=" + URLEncoder.encode("https://app.example.com/cb", UTF_8)
            + "&scope=openid+account.read"
            + "&state=test_state"
            + "&code_challenge=" + codeChallenge
            + "&code_challenge_method=S256";
        
        String authCode = simulateAuthorizationFlow(authUrl);
        
        ResponseEntity<Map> response = restTemplate.withBasicAuth("hesap-yoneticisi", "secret")
            .postForEntity("/oauth2/token",
                Map.of(
                    "grant_type", "authorization_code",
                    "code", authCode,
                    "redirect_uri", "https://app.example.com/cb",
                    "code_verifier", codeVerifier
                ), Map.class);
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        Map body = response.getBody();
        assertThat(body).containsKeys("access_token", "id_token", "refresh_token");
        
        DecodedJWT access = JWT.decode((String) body.get("access_token"));
        assertThat(access.getAudience()).contains("hesap-yoneticisi");
        assertThat(access.getClaim("scope").asString()).contains("account.read");
    }
    
    @Test
    void shouldRejectAuthCodeWithoutPkceVerifier() {
        ResponseEntity<Map> response = restTemplate.withBasicAuth("hesap-yoneticisi", "secret")
            .postForEntity("/oauth2/token",
                Map.of(
                    "grant_type", "authorization_code",
                    "code", "valid_code",
                    "redirect_uri", "https://app.example.com/cb"
                ), Map.class);
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
    }
    
    @Test
    void clientCredentialsFlow() {
        ResponseEntity<Map> response = restTemplate.withBasicAuth("payment-service", "secret")
            .postForEntity("/oauth2/token",
                Map.of("grant_type", "client_credentials",
                       "scope", "internal.transfer.write"), Map.class);
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        DecodedJWT token = JWT.decode((String) response.getBody().get("access_token"));
        assertThat(token.getSubject()).isEqualTo("payment-service");
    }
    
    @Test
    void shouldRejectScopeNotInRegisteredClient() {
        String authUrl = "/oauth2/authorize"
            + "?response_type=code"
            + "&client_id=hesap-yoneticisi"
            + "&scope=transfer.write"
            + "...";
        
        ResponseEntity<String> response = restTemplate.getForEntity(authUrl, String.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
    }
}
```

---

## Claude-verify prompt

```
OAuth2 + OIDC implementation'ımı banking-grade kriterlere göre değerlendir:

1. Authorization Server:
   - Spring Authorization Server kullanıyor mu?
   - /.well-known/openid-configuration reachable?
   - JWKS endpoint key rotation destekli?

2. Grant types:
   - Authorization Code + PKCE (mandatory)?
   - Client Credentials (service-to-service)?
   - Refresh Token rotation?
   - Implicit / Resource Owner Password DEPRECATED yok?

3. PKCE:
   - requireProofKey(true) registered client'ta?
   - S256 challenge method?
   - Mobile/SPA için mandatory?

4. Scopes:
   - Fine-grained banking scope'lar (account.read, transfer.write)?
   - Least privilege?
   - Consent screen scope listele?

5. Token TTL:
   - Access 5-15 dakika?
   - Refresh 1-24 saat?
   - reuseRefreshTokens(false) — rotation?

6. OIDC:
   - openid scope ID token trigger?
   - ID token claim'leri (sub, aud, exp, iat, nonce)?
   - UserInfo endpoint çalışıyor mu?

7. Client config:
   - Public client (mobile) → client_secret YOK + PKCE şart?
   - Confidential client → client_secret + (optional) mTLS?
   - Exact redirect_uri match (wildcard YOK)?

8. State + nonce:
   - state parameter CSRF prevention?
   - nonce replay prevention?

9. FAPI alignment:
   - HTTPS mandatory?
   - Audience strict validation?
   - mTLS (banking üst seviye)?

10. Anti-pattern:
    - Implicit / Password grant YOK?
    - Wildcard redirect_uri YOK?
    - Long access TTL YOK?
    - PKCE'siz public client YOK?
    - state'siz authorize YOK?
```

---

## Tamamlama kriterleri

- [ ] Spring Authorization Server kurulu + discovery reachable
- [ ] RegisteredClient (auth code + client credentials)
- [ ] PKCE flow end-to-end test
- [ ] Refresh rotation
- [ ] OIDC scope (openid, profile, email)
- [ ] ID token decode + claim doğrulama
- [ ] Client app (oauth2Login) third-party flow
- [ ] Custom consent screen + audit
- [ ] 4+ scope (banking fine-grained)
- [ ] Token introspection
- [ ] RP-initiated logout

---

## Defter notları (10 madde)

1. "OAuth 2.0 authorization vs OIDC authentication ayrımı: ____."
2. "Authorization Code + PKCE flow 7 adım: ____."
3. "Client Credentials banking internal service-to-service: ____."
4. "PKCE code_verifier + code_challenge anti-intercept: ____."
5. "ID token vs Access token banking kullanım farkı: ____."
6. "Refresh token rotation `reuseRefreshTokens(false)`: ____."
7. "Discovery endpoint `/.well-known/openid-configuration` içeriği: ____."
8. "FAPI Advanced + mTLS + JAR banking adoption (BDDK): ____."
9. "OAuth 2.1 deprecated grant'ler (Implicit, Password) sebebi: ____."
10. "Banking fine-grained scope tasarımı + consent UI audit: ____."
