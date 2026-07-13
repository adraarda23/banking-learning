# Topic 8.5 — Keycloak: Production-grade IdP for Banking

## Hedef

Keycloak'u banking-grade identity provider olarak kurmak, konfigüre etmek, Spring Boot ile entegre etmek. Realm/client/role/group hierarchy, federation (LDAP/AD), social login, MFA, customizing themes, audit, performance tuning, HA deployment.

## Süre

Okuma: 2 saat • Mini task: 3 saat • Test: 1 saat • Toplam: ~6 saat

## Önbilgi

- Topic 8.4 (OAuth2/OIDC) bitti
- Docker, basic K8s
- LDAP/AD temel (federation için)

---

## Kavramlar

### 1. Keycloak — niye?

**Build vs Buy:**
- Spring Authorization Server build → tüm UI/admin/audit/federation **kendi**
- Keycloak ready: realm, user, role, MFA, social, federation, audit, themes — **out-of-box**

**Banking pratiği:**
- Internal: Çoğunlukla **Keycloak** veya **ForgeRock**, **Okta**, **Ping Identity**, **Auth0**
- Bazı bankalar: **kendi yapımları** (legacy)
- Mid-size: **Keycloak** yaygın (open source, on-prem deployment)

**Keycloak özellikleri:**
- OAuth 2.0 + OIDC + SAML 2.0
- LDAP/AD federation
- Social login (Google, Facebook, Apple)
- MFA (TOTP, WebAuthn, SMS via extension)
- Themes (login UI customize)
- Admin UI + REST API
- Multi-realm (multi-tenant)
- Cluster mode (HA, Infinispan replicate)
- Backup-restore
- Audit events

### 2. Keycloak hierarchy

```
Keycloak server
├── Realm: master            (admin)
├── Realm: banking           ← banking için
│   ├── Users                (banking customers + employees)
│   ├── Groups
│   │   ├── customers
│   │   ├── tellers
│   │   └── admins
│   ├── Roles                (realm roles)
│   │   ├── customer
│   │   ├── teller
│   │   └── admin
│   ├── Clients
│   │   ├── banking-mobile   (public client, PKCE)
│   │   ├── banking-web      (public client, PKCE)
│   │   ├── banking-api      (resource, bearer-only)
│   │   ├── teller-app       (confidential)
│   │   └── payment-service  (service account, client credentials)
│   ├── Identity Providers   (Google, AD)
│   ├── Authentication Flows (login, MFA flow)
│   └── User Federation      (LDAP)
└── Realm: tenant-corp       (corporate banking)
```

**Realm** = isolated tenant. Banking için tek realm yeterli (multi-realm corporate vs retail bölünebilir).

### 3. Local kurulum — Docker

```bash
docker run -d --name keycloak \
  -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:24.0 \
  start-dev
```

Production:
```bash
docker run -d --name keycloak \
  -p 8443:8443 \
  -e KC_DB=postgres \
  -e KC_DB_URL=jdbc:postgresql://db:5432/keycloak \
  -e KC_DB_USERNAME=keycloak \
  -e KC_DB_PASSWORD=secret \
  -e KC_HOSTNAME=auth.mavibank.com \
  -e KC_HTTPS_CERTIFICATE_FILE=/etc/x509/https/tls.crt \
  -e KC_HTTPS_CERTIFICATE_KEY_FILE=/etc/x509/https/tls.key \
  quay.io/keycloak/keycloak:24.0 \
  start --optimized
```

Admin console: `https://auth.mavibank.com/admin/`

### 4. Realm creation — banking

Admin UI → "Add realm" → "banking".

Realm settings:
- **Login**: User registration ON (banking için), Email as username ON
- **Email**: SMTP config (verification için)
- **Themes**: Login theme = "banking-custom"
- **Tokens**: 
  - Access token lifespan: 15 min
  - Refresh token lifespan: 60 min
  - SSO session idle: 30 min
  - SSO session max: 8 hours
- **Security defenses**:
  - Brute force detection: ON
  - Max login failures: 5
  - Wait time: 15 min
  - Permanent lockout: OFF (banking için manual review)
- **General**:
  - SSL required: external requests (production)

### 5. Client setup — banking-mobile (public, PKCE)

Admin UI → Clients → Create.

```
Client type: OpenID Connect
Client ID: banking-mobile
Name: Maven Bank Mobile App
Always display in console: ON
Client authentication: OFF   (public client)
Authorization: OFF
Authentication flow:
  ✓ Standard flow
  ☐ Direct access grants   (banking: KAPALI)
  ☐ Implicit flow          (KAPALI)
  ☐ Service account roles
Valid redirect URIs: 
  com.mavibank.app://callback
  https://app.mavibank.com/callback
Valid post logout redirect URIs:
  com.mavibank.app://logout
Web origins: +
Advanced:
  Proof Key for Code Exchange: S256
  Access Token Lifespan: 15 min
  Client Session Idle: 30 min
```

### 6. Client setup — banking-api (resource server)

```
Client ID: banking-api
Client authentication: ON
Authorization: ON   (banking için fine-grained)
Authentication flow:
  ☐ Standard flow
  ☐ Direct access grants
  ☐ Implicit flow
  ✓ Service accounts roles  (optional, for client_credentials)
```

Resource server JWT validation: `issuer-uri` ile otomatik.

### 7. Client setup — payment-service (client credentials)

```
Client ID: payment-service
Client authentication: ON
Authorization: OFF
Authentication flow:
  ☐ Standard flow
  ☐ Direct access grants
  ✓ Service account roles   (client_credentials için)
Credentials → Secret: <copy>
Service Account Roles → Assign:
  - banking-api: internal.account.read
  - banking-api: internal.transfer.write
```

### 8. Roles — realm vs client

**Realm role:** Cross-client. `customer`, `teller`, `admin`.

**Client role:** Client-specific. `banking-api: account.read`, `banking-api: transfer.write`.

**Composite role:** Other role'leri içerir.

```
Realm role "customer":
  - banking-api: account.read
  - banking-api: transactions.read
  - banking-api: profile.read

Realm role "teller":
  composite of: customer + banking-api: card.write + banking-api: transfer.write
```

Banking için: **realm role** + **client role** kombine.

### 9. Groups — banking user organization

```
Groups:
├── customers
│   ├── retail
│   └── corporate
├── employees
│   ├── tellers
│   ├── managers
│   └── admins
```

Group'a role assign → her üye'ye otomatik.

Yeni teller → "employees/tellers" group'una koy → otomatik teller role.

### 10. Users + attributes

User attributes — banking custom data:

```
User: ahmet.yilmaz@mavibank.com
Attributes:
  tenant: TR
  branch: istanbul-1
  customer_id: cust-456
  mfa_enabled: true
  kvkk_consent: 2024-01-15
```

Bu attribute'lar JWT'ye claim olarak inject edilebilir (token mapper).

### 11. Token mapper — custom claims

Client → Client scopes → banking-claims → Add mapper.

```
Name: tenant-mapper
Mapper Type: User Attribute
User Attribute: tenant
Token Claim Name: tenant
Claim JSON Type: String
Add to ID token: ON
Add to access token: ON
Add to userinfo: ON
```

Sonuç JWT:
```json
{
  "sub": "user-123",
  "tenant": "TR",
  "branch": "istanbul-1",
  ...
}
```

### 12. Authentication flows — banking MFA

Admin → Authentication → Flows.

**Browser flow** (default):
```
1. Cookie
2. Identity Provider Redirector
3. Forms (login/password)
   - Browser Forms (subflow)
     - Username Password Form  [REQUIRED]
     - Conditional OTP         [CONDITIONAL]
       - User Configured        [CONDITIONAL]
       - OTP Form               [REQUIRED]
```

**Banking custom flow:**

1. Copy "Browser" → "Banking-MFA-Browser"
2. Subflows:
   ```
   Banking-MFA-Browser
   ├── Cookie [ALTERNATIVE]
   ├── Forms [ALTERNATIVE]
   │   ├── Username Password Form [REQUIRED]
   │   ├── Banking Risk Assessment [REQUIRED]    ← custom SPI
   │   │   (IP reputation, device fingerprint)
   │   ├── Conditional OTP [REQUIRED]
   │   └── Conditional WebAuthn [CONDITIONAL]   ← high-value
   ```
3. Realm settings → Login flow → "Banking-MFA-Browser"

**Custom SPI (Service Provider Interface):**

```java
public class BankingRiskAuthenticator implements Authenticator {
    
    @Override
    public void authenticate(AuthenticationFlowContext context) {
        String ip = context.getConnection().getRemoteAddr();
        String userAgent = context.getHttpRequest().getHttpHeaders()
            .getRequestHeader("User-Agent").get(0);
        UserModel user = context.getUser();
        
        int riskScore = riskService.evaluate(user, ip, userAgent);
        
        if (riskScore < 30) {
            context.success();   // Low risk, skip extra MFA
        } else if (riskScore < 70) {
            context.attempted();   // Force OTP
        } else {
            context.failure(AuthenticationFlowError.ACCESS_DENIED);
            // Alert security
            securityOps.alert("High-risk login: " + user.getUsername());
        }
    }
    
    @Override
    public boolean requiresUser() { return true; }
    
    @Override
    public boolean configuredFor(KeycloakSession session, RealmModel realm, UserModel user) {
        return true;
    }
}
```

Build → Keycloak `providers/` dizinine kopyala → restart.

### 13. LDAP/AD federation

Banking internal user'ları AD'de.

Admin → User Federation → Add → LDAP.

```
Vendor: Active Directory
Connection URL: ldaps://ad.mavibank.com:636
Bind Type: simple
Bind DN: CN=keycloak-svc,OU=ServiceAccounts,DC=mavibank,DC=com
Bind Credential: <password>
Edit Mode: READ_ONLY   (AD master)
Users DN: OU=Employees,DC=mavibank,DC=com
Username LDAP attribute: sAMAccountName
RDN LDAP attribute: cn
UUID LDAP attribute: objectGUID
User Object Classes: person, organizationalPerson, user
Connection Pooling: ON
Pagination: ON
Allow Kerberos: ON   (SSO için)
Sync Settings:
  Periodic Full Sync: every 1 hour
  Periodic Changed Users Sync: every 5 min
```

LDAP mapper'lar (group import, role mapping).

Sonuç: Banking employee AD credentials ile Keycloak'a login. Realm'de user yansır.

### 14. Spring Boot entegrasyon

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.mavibank.com/realms/banking
          jwk-set-uri: https://auth.mavibank.com/realms/banking/protocol/openid-connect/certs
      client:
        registration:
          banking-keycloak:
            client-id: banking-web
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/banking-keycloak"
            scope: openid,profile,email
        provider:
          banking-keycloak:
            issuer-uri: https://auth.mavibank.com/realms/banking
```

```java
@Configuration
@EnableMethodSecurity
public class KeycloakConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(a -> a
                .requestMatchers("/admin/**").hasRole("admin")
                .requestMatchers("/teller/**").hasRole("teller")
                .anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt(j -> 
                j.jwtAuthenticationConverter(keycloakJwtConverter())));
        return http.build();
    }
    
    private JwtAuthenticationConverter keycloakJwtConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            Set<GrantedAuthority> authorities = new HashSet<>();
            
            Map<String, Object> realmAccess = jwt.getClaim("realm_access");
            if (realmAccess != null) {
                @SuppressWarnings("unchecked")
                List<String> roles = (List<String>) realmAccess.get("roles");
                if (roles != null) {
                    roles.forEach(r -> authorities.add(
                        new SimpleGrantedAuthority("ROLE_" + r)));
                }
            }
            
            Map<String, Object> resourceAccess = jwt.getClaim("resource_access");
            if (resourceAccess != null) {
                @SuppressWarnings("unchecked")
                Map<String, Object> bankingApi = (Map<String, Object>) resourceAccess.get("banking-api");
                if (bankingApi != null) {
                    @SuppressWarnings("unchecked")
                    List<String> clientRoles = (List<String>) bankingApi.get("roles");
                    if (clientRoles != null) {
                        clientRoles.forEach(r -> authorities.add(
                            new SimpleGrantedAuthority("CLIENT_" + r)));
                    }
                }
            }
            
            return authorities;
        });
        return converter;
    }
}
```

Controller:

```java
@RestController
public class TransferController {
    
    @PostMapping("/transfers")
    @PreAuthorize("hasAuthority('CLIENT_transfer.write')")
    public Transfer transfer(@RequestBody TransferRequest req,
                            @AuthenticationPrincipal Jwt jwt) {
        UUID userId = UUID.fromString(jwt.getSubject());
        String tenant = jwt.getClaimAsString("tenant");
        String branch = jwt.getClaimAsString("branch");
        return transferService.transfer(req, userId, tenant);
    }
}
```

### 15. Audit + events

Admin → Events → Config.

```
Save Events: ON
Event types to save: LOGIN, LOGOUT, LOGIN_ERROR, REFRESH_TOKEN, 
                     UPDATE_PASSWORD, UPDATE_PROFILE
Expiration: 90 days   (regulatory)

Admin Events Settings:
Save events: ON
Include representation: ON   (full change details)
```

Banking için Keycloak event listener SPI ile:
- Login event → Kafka topic
- High-value action → SIEM
- Failed login + threshold → security ops alert

### 16. Theme — banking branding

Keycloak default theme banking için yetersiz (logo, color, language).

```
themes/banking/
├── login/
│   ├── login.ftl                  (Freemarker)
│   ├── messages/
│   │   ├── messages_tr.properties
│   │   └── messages_en.properties
│   ├── resources/
│   │   ├── css/styles.css
│   │   ├── img/logo.svg
│   │   └── js/
│   └── theme.properties
├── email/
│   ├── html/
│   │   └── email-verification.ftl
│   └── messages/
│       └── messages_tr.properties
└── account/
```

Realm settings → Themes → Login Theme: "banking".

Banking için: KVKK consent, banking güvenlik mesajları, brand identity, Türkçe-İngilizce.

### 17. HA deployment

```
[Load Balancer]
    ↓
[Keycloak Node 1] ←→ [Infinispan cluster] ←→ [Keycloak Node 2]
    ↓                                              ↓
    └──────────→ [PostgreSQL] (primary + replica) ←┘
```

K8s manifest:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: keycloak
spec:
  serviceName: keycloak-headless
  replicas: 3
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
      - name: keycloak
        image: quay.io/keycloak/keycloak:24.0
        args: ["start", "--optimized", "--cache-stack=kubernetes"]
        env:
        - name: KC_DB
          value: postgres
        - name: KC_DB_URL
          value: jdbc:postgresql://postgres:5432/keycloak
        - name: JAVA_OPTS_APPEND
          value: "-Djgroups.dns.query=keycloak-headless.banking.svc.cluster.local"
        - name: KC_HOSTNAME
          value: auth.mavibank.com
        - name: KC_HTTPS_CERTIFICATE_FILE
          value: /etc/tls/tls.crt
        - name: KC_HTTPS_CERTIFICATE_KEY_FILE
          value: /etc/tls/tls.key
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
        resources:
          requests:
            memory: 1Gi
            cpu: 500m
          limits:
            memory: 2Gi
            cpu: 2
```

### 18. Performance tuning

- **DB connection pool**: `KC_DB_POOL_MIN_SIZE=10`, `KC_DB_POOL_MAX_SIZE=50`
- **JVM heap**: `-Xms1g -Xmx2g`
- **Cache**: Infinispan distributed-cache for session
- **JWKS cache**: Resource server side cache (Spring Security default 5 min)
- **Build-time optimization**: `kc.sh build` ile features precompile
- **Theme caching**: Production'da theme cache ON

Benchmark hedef: Login 200ms p99, token exchange 50ms p99.

### 19. Banking anti-pattern'leri

**Anti-pattern 1: Direct access grants (password) ENABLED**

Banking mobile/web public client'larda password grant açık → OAuth 2.1 deprecated, third-party password öğrenir. **KAPAT.**

**Anti-pattern 2: Implicit flow ENABLED**

URL fragment'inde token → log/history leak. **KAPAT.**

**Anti-pattern 3: PKCE OFF mobile/SPA için**

Mobile public client → PKCE şart. Client settings → Advanced → Proof Key for Code Exchange: S256.

**Anti-pattern 4: Wildcard redirect URI**

Subdomain takeover. **Exact match.**

**Anti-pattern 5: Master realm production'da**

Master realm sadece admin. Banking için ayrı realm. Master'a customer koyma.

**Anti-pattern 6: Admin password default**

`admin/admin`. Production: strong random + IP allowlist + MFA admin için.

**Anti-pattern 7: HTTP allowed production'da**

SSL required: external requests / all (production).

**Anti-pattern 8: User federation read-write yanlış**

AD master → Keycloak `READ_ONLY`. User Keycloak'tan password değiştiremez (AD'de değiştirsin).

**Anti-pattern 9: Brute force detection OFF**

Banking için ON şart. Max failures 5, wait 15 min.

**Anti-pattern 10: Event listener yok**

Banking audit/SIEM için event listener şart. Without listener: compliance gap.

---

## Önemli olabilecek araştırma kaynakları

- Keycloak Server Administration Guide
- Keycloak Securing Apps and Services
- Keycloak Developer Guide (SPI)
- Spring Security OAuth2 Keycloak integration
- BDDK kimlik yönetimi requirements
- Keycloak community forums

---

## Mini task'ler

### Task 8.5.1 — Docker Keycloak + PostgreSQL (30 dk)

Docker compose ile Keycloak + Postgres ayağa kaldır. Admin'e login.

### Task 8.5.2 — Realm + clients (60 dk)

`banking` realm. `banking-web` (public + PKCE), `banking-api` (resource), `payment-service` (service account), `teller-app` (confidential).

### Task 8.5.3 — Roles + groups + user (45 dk)

`customer`, `teller`, `admin` realm roles. `customers/retail`, `employees/tellers` groups. 3 test user.

### Task 8.5.4 — Token mapper banking attributes (30 dk)

User attributes `tenant`, `branch` → access token claim. JWT decode → confirm.

### Task 8.5.5 — Custom authentication flow + MFA (60 dk)

"Banking-MFA-Browser" flow. OTP enforced. Test: Yeni login → OTP setup screen.

### Task 8.5.6 — Spring Boot resource server (60 dk)

`oauth2ResourceServer.jwt()` config. `keycloakJwtConverter` realm role + client role mapping. `@PreAuthorize` test.

### Task 8.5.7 — Spring Security OAuth2 Client + login (45 dk)

`oauth2Login()` ile Keycloak login flow.

### Task 8.5.8 — Event listener (60 dk)

Custom event listener SPI → Login event Kafka'ya. Banking audit.

### Task 8.5.9 — Banking theme (60 dk)

Custom login theme: logo, Türkçe, banking colors, KVKK consent.

### Task 8.5.10 — Brute force test (30 dk)

5 yanlış password → user locked 15 min. Sonra unlock.

---

## Test yazma rehberi

```java
@SpringBootTest
@Testcontainers
class KeycloakIntegrationTest {
    
    @Container
    static KeycloakContainer keycloak = new KeycloakContainer("quay.io/keycloak/keycloak:24.0")
        .withRealmImportFile("/banking-realm.json");
    
    @DynamicPropertySource
    static void registerKeycloakProps(DynamicPropertyRegistry registry) {
        registry.add("spring.security.oauth2.resourceserver.jwt.issuer-uri",
            () -> keycloak.getAuthServerUrl() + "/realms/banking");
    }
    
    @Test
    void shouldAuthenticateWithKeycloakToken() throws Exception {
        String token = obtainKeycloakToken("ahmet", "password", "banking-web");
        
        mockMvc.perform(get("/v1/accounts/me")
            .header("Authorization", "Bearer " + token))
            .andExpect(status().isOk());
    }
    
    @Test
    void shouldRejectInsufficientRole() throws Exception {
        String customerToken = obtainKeycloakToken("ahmet", "password", "banking-web");
        
        mockMvc.perform(get("/admin/users")
            .header("Authorization", "Bearer " + customerToken))
            .andExpect(status().isForbidden());
    }
    
    @Test
    void shouldIncludeBankingClaimsInToken() throws Exception {
        String token = obtainKeycloakToken("ahmet", "password", "banking-web");
        DecodedJWT decoded = JWT.decode(token);
        
        assertThat(decoded.getClaim("tenant").asString()).isEqualTo("TR");
        assertThat(decoded.getClaim("branch").asString()).isEqualTo("istanbul-1");
    }
    
    @Test
    void shouldLockUserAfter5FailedAttempts() {
        for (int i = 0; i < 5; i++) {
            assertThatThrownBy(() -> 
                obtainKeycloakToken("ahmet", "wrong-password", "banking-web"))
                .hasMessageContaining("invalid_grant");
        }
        
        // 6th attempt with CORRECT password should still fail
        assertThatThrownBy(() ->
            obtainKeycloakToken("ahmet", "password", "banking-web"))
            .hasMessageContaining("Account is not fully set up");
    }
    
    @Test
    void clientCredentialsFlow_paymentService() throws Exception {
        String token = obtainClientCredentialsToken("payment-service", "secret");
        
        DecodedJWT decoded = JWT.decode(token);
        assertThat(decoded.getSubject()).startsWith("service-account-payment-service");
    }
}
```

---

## Claude-verify prompt

```
Keycloak setup'ımı banking-grade kriterlere göre değerlendir:

1. Kurulum:
   - PostgreSQL backend (production)?
   - HTTPS + cert config?
   - HA cluster mode (Infinispan)?
   - Health check endpoint?

2. Realm:
   - banking realm (master KULLANILMIYOR)?
   - SSL required: external/all?
   - Brute force detection ON (5/15)?
   - Token TTL (access 15 min, refresh 60 min)?

3. Client config:
   - banking-mobile public + PKCE S256?
   - Direct access grants OFF?
   - Implicit OFF?
   - Exact redirect URI (no wildcard)?
   - banking-api resource server?
   - payment-service service account + role assign?

4. Roles + groups:
   - Realm roles (customer, teller, admin)?
   - Client roles (banking-api: ...)?
   - Composite roles banking için?
   - Groups (customers/retail, employees/tellers)?

5. User attributes + token mapper:
   - tenant, branch attribute → JWT claim?
   - Token mapper ID + access token'a inject?

6. Authentication flow:
   - Custom banking MFA flow?
   - OTP enforced?
   - Conditional WebAuthn high-value?
   - Risk-based authentication (custom SPI)?

7. LDAP/AD federation:
   - READ_ONLY (AD master)?
   - Periodic sync (1h full, 5min changed)?
   - Group import?

8. Audit:
   - Event listener kafka/SIEM'e push?
   - LOGIN, LOGOUT, LOGIN_ERROR save?
   - 90-day retention?
   - Admin events full representation?

9. Spring integration:
   - oauth2ResourceServer.jwt() + issuer-uri?
   - JwtAuthenticationConverter realm + client role mapping?
   - @PreAuthorize role check?

10. Anti-pattern:
    - Direct access grants OFF?
    - Implicit OFF?
    - PKCE ON mobile/SPA?
    - Wildcard redirect URI YOK?
    - HTTP allowed YOK?
    - Master realm customer YOK?
    - Brute force OFF YOK?
    - Event listener YOK durumu YOK?
```

---

## Tamamlama kriterleri

- [ ] Keycloak Docker + Postgres + HTTPS
- [ ] `banking` realm + 4 client
- [ ] 3 realm role + 4 client role + composite
- [ ] Groups hierarchy + user assign
- [ ] Token mapper (tenant, branch)
- [ ] Custom MFA flow + OTP
- [ ] LDAP/AD federation (mock veya gerçek)
- [ ] Event listener SPI → Kafka
- [ ] Banking custom theme
- [ ] Spring Boot resource server integration
- [ ] Spring Boot OAuth2 client (oauth2Login)
- [ ] 5+ integration test (Testcontainers)
- [ ] Brute force lockout test

---

## Defter notları (10 madde)

1. "Keycloak vs build-your-own (Spring Authorization Server) trade-off banking: ____."
2. "Realm + client + role + group hierarchy banking modeli: ____."
3. "Public client PKCE + confidential client secret farkı: ____."
4. "Realm role vs client role banking örnek + composite: ____."
5. "User attribute → token mapper → JWT claim flow: ____."
6. "Custom authentication flow + SPI risk assessment: ____."
7. "LDAP/AD federation READ_ONLY banking employee: ____."
8. "Event listener + Kafka/SIEM banking audit: ____."
9. "HA cluster Infinispan + PostgreSQL deployment: ____."
10. "Banking anti-pattern (direct access grants, wildcard URI, default admin): ____."
