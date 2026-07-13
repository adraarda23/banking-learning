# Topic 8.1 — Spring Security 6 Architecture: Filter Chain, Method Security

## Hedef

Spring Security 6'nın **filter chain** mimarisini içselleştirmek. Modern (Spring Boot 3+) **component-based** konfigürasyonu yazabilmek (`WebSecurityConfigurerAdapter` deprecated, `SecurityFilterChain` bean tabanlı). `AuthenticationManager`, `AuthenticationProvider`, `UserDetailsService`, `GrantedAuthority`, `AccessDecisionManager` zincirini bilmek. Method security (`@PreAuthorize`, `@PostAuthorize`, `@Secured`, `@RolesAllowed`) ile **iş kuralı düzeyinde** yetkilendirme yapabilmek. Banking için **"müşteri sadece kendi hesabını görür"** kuralını SpEL ile yazabilmek.

## Süre

Okuma: 2 saat • Mini task: 2.5 saat • Test: 1 saat • Toplam: ~5.5 saat

## Önbilgi

- Spring Boot 3, Spring MVC, REST controller temel
- HTTP request/response, header, status code
- Hexagonal architecture (Faz 1)
- Banking domain (Account, OwnerId)

---

## Kavramlar

### 1. Spring Security ne yapar — büyük resim

Spring Security 4 sorunu çözer:

1. **Authentication** — "Sen kimsin?" Kullanıcının kim olduğunu doğrular (password, token, certificate).
2. **Authorization** — "Ne yapabilirsin?" Authenticated kullanıcının belirli kaynağa/işleme yetkisi var mı?
3. **Protection against common attacks** — CSRF, session fixation, clickjacking, secure header'lar.
4. **Integration** — OAuth2, OIDC, SAML, LDAP, JWT gibi standartlarla.

**Banking için olmazsa olmaz**:

- Bir HTTP isteği gelir → **Spring Security önce dokunur** → kim olduğu belli olur → yetki kontrolü → controller'a ulaşır
- Eğer Spring Security'yi atlatırsan, controller doğrudan request alır = **production'da kelepçeli durmazsın**

### 2. Spring Security 6 ile gelen değişiklikler (Spring Boot 3)

Eski kod (Spring Security 5.7'den önce):

```java
// ❌ DEPRECATED in Spring Security 5.7, REMOVED in Spring Security 6.x
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
            .antMatchers("/public/**").permitAll()
            .anyRequest().authenticated()
            .and().csrf().disable();
    }
}
```

Yeni kod (Spring Security 6+):

```java
// ✅ MODERN — component-based, bean tabanlı
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .csrf(AbstractHttpConfigurer::disable);   // Banking REST API için sıklıkla
        return http.build();
    }
}
```

**Farklar:**

| Eski | Yeni |
|---|---|
| `WebSecurityConfigurerAdapter` extend | `SecurityFilterChain` bean'ı |
| `antMatchers` | `requestMatchers` |
| `authorizeRequests` | `authorizeHttpRequests` |
| Chain method `.and()` | Lambda DSL |
| `configure(HttpSecurity)` | Bean factory method |

**Neden değişti:** Lambda DSL daha okunabilir, configuration daha test edilebilir, multiple security chain (REST API + Admin Panel) tek class'ta tanımlanabilir.

### 3. Filter Chain — Spring Security'nin kalbi

Spring Security aslında bir **Servlet Filter** zinciridir. HTTP request geldiğinde:

```
HTTP Request
    ↓
┌─────────────────────────────────────────────────────────────┐
│  DelegatingFilterProxy (Servlet container'a kayıtlı)        │
│      ↓                                                       │
│  FilterChainProxy (Spring Security entry point)              │
│      ↓                                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ SecurityFilterChain (your config)                   │    │
│  │                                                     │    │
│  │  1. DisableEncodeUrlFilter                          │    │
│  │  2. WebAsyncManagerIntegrationFilter                │    │
│  │  3. SecurityContextHolderFilter                     │    │
│  │  4. HeaderWriterFilter                              │    │
│  │  5. CsrfFilter                                      │    │
│  │  6. LogoutFilter                                    │    │
│  │  7. BearerTokenAuthenticationFilter   ← JWT here    │    │
│  │  8. UsernamePasswordAuthenticationFilter            │    │
│  │  9. RequestCacheAwareFilter                         │    │
│  │ 10. SecurityContextHolderAwareRequestFilter         │    │
│  │ 11. AnonymousAuthenticationFilter                   │    │
│  │ 12. SessionManagementFilter                         │    │
│  │ 13. ExceptionTranslationFilter                      │    │
│  │ 14. AuthorizationFilter   ← @PreAuthorize evaluates │    │
│  └─────────────────────────────────────────────────────┘    │
│      ↓                                                       │
└─────────────────────────────────────────────────────────────┘
    ↓
DispatcherServlet → Controller
```

Her filter:
- Kendi sorumluluğunu çözer (örn. `CsrfFilter` CSRF token doğrular)
- `chain.doFilter(req, res)` ile **bir sonrakine** geçirir
- Veya hata varsa kısa devre yapar (`HTTP 401`, `403` döner)

**Banking implikasyonu:** Bir request controller'a ulaşmadan önce 10+ filtreden geçer. **"Authentication geç, authorization sıkı"** mantığı: kim olduğun erkenden tespit edilir, yetkin son anda kontrol edilir.

### 4. SecurityContext ve SecurityContextHolder

`SecurityContext` = mevcut authenticated kullanıcının bilgisi:

```java
public interface SecurityContext extends Serializable {
    Authentication getAuthentication();
    void setAuthentication(Authentication authentication);
}
```

`SecurityContextHolder` = thread-local storage:

```java
// Bir filter authentication yaptıktan sonra SecurityContext set eder
Authentication auth = new UsernamePasswordAuthenticationToken(principal, null, authorities);
SecurityContextHolder.getContext().setAuthentication(auth);

// Sonraki kodlar (controller, service) her yerden okuyabilir
Authentication current = SecurityContextHolder.getContext().getAuthentication();
String username = current.getName();
```

**ThreadLocal tuzağı:**

- Her HTTP isteği bir thread'de işlenir → SecurityContext o thread'e bağlı
- `@Async` veya yeni thread oluştururken **SecurityContext kopyalanmaz** → `MODE_INHERITABLETHREADLOCAL` veya `DelegatingSecurityContextRunnable` gerekir
- Banking örnek: bir `@Async` audit log writer thread'inde "kim audit yapıyor" bilgisi kaybolur

Modern alternatif: `Authentication` Spring 6 ile `@AuthenticationPrincipal` parametre olarak inject edilebilir, daha temiz:

```java
@GetMapping("/me")
public AccountResponse getMyAccount(@AuthenticationPrincipal Jwt jwt) {
    String customerId = jwt.getSubject();
    return service.findByOwner(customerId);
}
```

### 5. Authentication — kim olduğunu nasıl ispat ediyorsun

`Authentication` interface'i bir kullanıcının kimliğini temsil eder:

```java
public interface Authentication extends Principal, Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();   // roller, permission'lar
    Object getCredentials();                                    // password (auth sonrası null)
    Object getDetails();                                        // IP, session id...
    Object getPrincipal();                                      // user identity (genellikle UserDetails)
    boolean isAuthenticated();
    void setAuthenticated(boolean isAuthenticated);
    String getName();                                           // username
}
```

Concrete implementations:
- `UsernamePasswordAuthenticationToken` — username/password
- `BearerTokenAuthenticationToken` — JWT veya opaque token
- `OAuth2AuthenticationToken` — OAuth2 social login
- `PreAuthenticatedAuthenticationToken` — mTLS, header-based

**Authentication flow:**

```
1. Request gelir, AuthenticationFilter intercept eder
   (örn. UsernamePasswordAuthenticationFilter / BearerTokenAuthenticationFilter)
2. Filter request'ten credential extract eder (form param veya Authorization header)
3. Filter bir UNAUTHENTICATED Authentication oluşturur (principal=username, credentials=password)
4. AuthenticationManager.authenticate(authToken) çağrılır
5. AuthenticationManager (genellikle ProviderManager) AuthenticationProvider'ları sırayla dener
6. AuthenticationProvider:
   - DaoAuthenticationProvider → UserDetailsService.loadUserByUsername() çağırır
   - Password'u PasswordEncoder ile karşılaştırır
   - AUTHENTICATED Authentication döner (principal=UserDetails, credentials=null, authorities=[...])
7. Filter SecurityContextHolder.getContext().setAuthentication(result) yapar
8. chain.doFilter(req, res) → controller'a doğru ilerler
```

### 6. AuthenticationProvider, UserDetailsService, UserDetails

**`UserDetailsService`** — DB'den kullanıcı çek:

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

Banking implementation:

```java
@Service
public class BankingUserDetailsService implements UserDetailsService {
    
    private final CustomerRepository customerRepository;
    private final PasswordEncoder passwordEncoder;
    
    @Override
    public UserDetails loadUserByUsername(String username) {
        // username = TC No veya customer ID
        Customer customer = customerRepository.findByTcNo(username)
            .orElseThrow(() -> new UsernameNotFoundException("Müşteri bulunamadı"));
        
        if (customer.isLocked()) {
            throw new LockedException("Hesap kilitli");
        }
        if (!customer.isActive()) {
            throw new DisabledException("Hesap aktif değil");
        }
        
        return User.builder()
            .username(customer.getTcNo())
            .password(customer.getPasswordHash())
            .authorities(buildAuthorities(customer))
            .accountLocked(customer.isLocked())
            .disabled(!customer.isActive())
            .credentialsExpired(customer.isPasswordExpired())
            .build();
    }
    
    private Collection<? extends GrantedAuthority> buildAuthorities(Customer customer) {
        List<GrantedAuthority> authorities = new ArrayList<>();
        // ROLE_CUSTOMER, ROLE_PRIVATE_BANKING, vs.
        authorities.add(new SimpleGrantedAuthority("ROLE_" + customer.getSegment().name()));
        // Permission'lar
        customer.getPermissions().forEach(p -> 
            authorities.add(new SimpleGrantedAuthority(p.getCode())));
        return authorities;
    }
}
```

**`UserDetails`** — Spring Security'nin kullandığı kullanıcı kontratı:

```java
public interface UserDetails extends Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();
    String getPassword();
    String getUsername();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}
```

Spring'in built-in `User` class'ı bu interface'i implement eder (yukarıda kullandığımız).

**Önemli:** UserDetails dönen objeden **şifre alanı authentication sonrası null'lanmalı** (memory leak'i önlemek için). Spring built-in `eraseCredentials` bunu yapar.

**`AuthenticationProvider`** — gerçek authentication mantığı:

```java
public interface AuthenticationProvider {
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
    boolean supports(Class<?> authentication);
}
```

Spring Security'nin built-in provider'ları:
- `DaoAuthenticationProvider` — username/password (UserDetailsService kullanır)
- `JwtAuthenticationProvider` — JWT (Topic 8.3'te)
- `OAuth2LoginAuthenticationProvider` — OAuth2 social login

**Multiple provider** — birden fazla auth metodu desteklemek için:

```java
@Bean
AuthenticationManager authenticationManager(HttpSecurity http,
                                           UserDetailsService uds,
                                           PasswordEncoder encoder) throws Exception {
    AuthenticationManagerBuilder builder = http.getSharedObject(AuthenticationManagerBuilder.class);
    builder
        .userDetailsService(uds)
        .passwordEncoder(encoder);
    // İkinci provider eklemek istersen .authenticationProvider(...) zinciri
    return builder.build();
}
```

### 7. GrantedAuthority — yetki birimi

```java
public interface GrantedAuthority extends Serializable {
    String getAuthority();
}
```

Spring Security convention:
- `ROLE_*` prefix'i = **rol** (örn. `ROLE_ADMIN`, `ROLE_CUSTOMER`)
- Prefix'siz string = **permission** (örn. `account:read`, `transfer:execute`)

`hasRole("ADMIN")` çağrısı otomatik olarak `ROLE_ADMIN` arar (prefix'i ekler).
`hasAuthority("account:read")` çağrısı tam string arar.

**Banking role hierarchy örneği:**

```
ROLE_ADMIN
  ↓ extends
ROLE_BRANCH_MANAGER
  ↓ extends
ROLE_TELLER
  ↓ extends
ROLE_CUSTOMER
```

Hierarchy ile bir admin otomatik olarak teller olabilir:

```java
@Bean
RoleHierarchy roleHierarchy() {
    RoleHierarchyImpl hierarchy = new RoleHierarchyImpl();
    hierarchy.setHierarchy("""
        ROLE_ADMIN > ROLE_BRANCH_MANAGER
        ROLE_BRANCH_MANAGER > ROLE_TELLER
        ROLE_TELLER > ROLE_CUSTOMER
        """);
    return hierarchy;
}

@Bean
MethodSecurityExpressionHandler methodSecurityExpressionHandler(RoleHierarchy roleHierarchy) {
    var handler = new DefaultMethodSecurityExpressionHandler();
    handler.setRoleHierarchy(roleHierarchy);
    return handler;
}
```

### 8. AccessDecisionManager → AuthorizationManager (Spring Security 6)

Eski (deprecated):

```java
// Spring Security 5.x — AccessDecisionManager + AccessDecisionVoter
```

Yeni (Spring Security 6):

```java
public interface AuthorizationManager<T> {
    AuthorizationDecision check(Supplier<Authentication> authentication, T object);
}
```

Lambda DSL ile custom authorization:

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/v1/admin/**").hasRole("ADMIN")
    .requestMatchers(HttpMethod.GET, "/v1/accounts/{id}").access(
        AuthorizationManagers.allOf(
            AuthenticatedAuthorizationManager.authenticated(),
            ownerAuthorizationManager()
        )
    )
    .anyRequest().authenticated()
);

@Bean
AuthorizationManager<RequestAuthorizationContext> ownerAuthorizationManager() {
    return (authentication, context) -> {
        String accountId = context.getVariables().get("id");
        Authentication auth = authentication.get();
        String customerId = auth.getName();
        // Check from service if customerId owns accountId
        boolean owns = accountOwnershipService.isOwner(customerId, accountId);
        return new AuthorizationDecision(owns);
    };
}
```

### 9. Method Security — `@PreAuthorize`, `@PostAuthorize`, `@Secured`, `@RolesAllowed`

URL-based authorization yetersiz. **Method-level** authorization daha granular ve domain logic'e yakın.

Aktif etmek:

```java
@Configuration
@EnableMethodSecurity   // default: prePostEnabled = true
public class MethodSecurityConfig {
}
```

**`@PreAuthorize`** — method çağrılmadan **önce** çalışır:

```java
@Service
public class AccountQueryService {
    
    @PreAuthorize("hasRole('CUSTOMER')")
    public Account getById(AccountId id) {
        // ...
    }
    
    // SpEL ile parametre erişimi
    @PreAuthorize("hasRole('CUSTOMER') and #ownerId == authentication.name")
    public List<Account> listByOwner(String ownerId) {
        // ...
    }
    
    // SpEL ile bean çağrısı (BU GÜÇLÜ)
    @PreAuthorize("@accountAccessChecker.canAccess(authentication.name, #id)")
    public Account getAccount(AccountId id) {
        // ...
    }
}
```

**`@PostAuthorize`** — method çağrıldıktan **sonra** çalışır, **return value**'yu kontrol eder:

```java
// Return edilen Account'un owner'ı current user mı?
@PostAuthorize("returnObject.ownerId.value == authentication.name")
public Account getById(AccountId id) {
    return repository.findById(id).orElseThrow();
}
```

**Önemli:** `@PostAuthorize` method'u çalıştırır, sonra erişim reddederse exception fırlatır. **Side effect varsa tehlike** — DB'ye yazılmış olur. Bu yüzden çoğunlukla `@PreAuthorize` tercih edilir.

**`@PreFilter` ve `@PostFilter`** — collection elemanlarını filtreler:

```java
@PostFilter("filterObject.ownerId.value == authentication.name")
public List<Account> getAllAccounts() {
    return repository.findAll();
}
// Sadece current user'ın kendi account'ları döner
```

**Performans uyarısı:** `@PostFilter` tüm collection'ı DB'den çekip sonra filtreler. Banking'de bu N+1 katastrofi olabilir. Bunun yerine **DB query'sinde filter** uygula.

**`@Secured` ve `@RolesAllowed`** — daha eski, daha kısıtlı (sadece role listesi):

```java
@Secured({"ROLE_ADMIN", "ROLE_TELLER"})  // OR mantığı
public void closeAccount(AccountId id) { ... }

@RolesAllowed("ROLE_ADMIN")  // JSR-250 (jakarta.annotation)
public void deleteCustomer(CustomerId id) { ... }
```

Banking'de **`@PreAuthorize` standart** — daha güçlü, SpEL ile compliance kuralları yazılır.

### 10. SpEL — Spring Expression Language ile güç

`@PreAuthorize` içinde SpEL kullanılır. Banking-relevant ifadeler:

| Ifade | Anlamı |
|---|---|
| `authentication` | Mevcut `Authentication` objesi |
| `authentication.name` | Username (genellikle customerId / TC No) |
| `authentication.principal` | `UserDetails` veya `Jwt` |
| `principal` | shortcut for `authentication.principal` |
| `hasRole('ADMIN')` | Role check |
| `hasAuthority('account:read')` | Permission check |
| `hasAnyRole('ADMIN', 'TELLER')` | OR |
| `permitAll`, `denyAll` | Built-in |
| `#paramName` | Method parametresi |
| `#paramName.field` | Parametre field'ı |
| `returnObject` | (PostAuthorize'da) return value |
| `@beanName.method(...)` | Spring bean method çağrısı |

Banking örnekleri:

```java
// Müşteri kendi hesabını sorgulayabilir
@PreAuthorize("#ownerId == authentication.name or hasRole('ADMIN')")
List<Account> findByOwner(String ownerId);

// Transfer initiator hesap sahibi olmalı, OR teller olmalı
@PreAuthorize(
    "@accountAccessChecker.canTransferFrom(authentication.name, #request.fromAccountId) " +
    "or hasRole('TELLER')"
)
Transfer execute(TransferRequest request);

// Yüksek tutarlı işlem için ek role
@PreAuthorize(
    "#amount.amount().compareTo(T(java.math.BigDecimal).valueOf(50000)) < 0 " +
    "or hasRole('SENIOR_TELLER')"
)
void approveLargeTransfer(Money amount);
```

**Custom SpEL bean** banking için en güçlü pattern:

```java
@Component("accountAccessChecker")
public class AccountAccessChecker {
    
    private final AccountRepository accountRepository;
    private final AuditLogger auditLogger;
    
    public boolean canAccess(String customerId, String accountId) {
        boolean owns = accountRepository.findById(new AccountId(UUID.fromString(accountId)))
            .map(a -> a.getOwnerId().value().toString().equals(customerId))
            .orElse(false);
        
        if (!owns) {
            auditLogger.logUnauthorizedAccess(customerId, accountId);
        }
        return owns;
    }
    
    public boolean canTransferFrom(String customerId, String fromAccountId) {
        // Aynı + ek kurallar (örn. hesap aktif mi, daily limit aşıldı mı)
        return canAccess(customerId, fromAccountId);
    }
}
```

`@accountAccessChecker.canAccess(...)` ifadesi runtime'da Spring container'dan bean'i çeker ve method'unu çalıştırır.

### 11. Banking için "müşteri sadece kendi hesabı" pattern'i

Bu, banking back-end developer'ın yazacağı **en sık yetki kuralıdır**. Üç katmanda uygulanır:

**Katman 1 — Authorization filter (URL level):**

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/v1/accounts/{id}").access(ownerAccessManager())
    .requestMatchers("/v1/accounts").authenticated()
);
```

**Katman 2 — Method security (service level):**

```java
@Service
public class GetAccountService implements GetAccountUseCase {
    
    @PreAuthorize("@accountAccessChecker.canAccess(authentication.name, #id.value().toString())")
    public Account execute(AccountId id) {
        return repository.findById(id).orElseThrow(() -> new AccountNotFoundException(id));
    }
}
```

**Katman 3 — Domain query filter (DB level):**

```java
// Repository query her zaman ownerId ile birlikte
public interface AccountRepository {
    Optional<Account> findByIdAndOwner(AccountId id, OwnerId owner);
}

// Service:
@PreAuthorize("isAuthenticated()")
public Account execute(AccountId id, @AuthenticationPrincipal Jwt jwt) {
    OwnerId owner = new OwnerId(UUID.fromString(jwt.getSubject()));
    return repository.findByIdAndOwner(id, owner)
        .orElseThrow(() -> new AccountNotFoundException(id));  // Aynı 404 — enumeration koruması
}
```

**3 katman defense-in-depth:**
- Filter URL'i kaba savar
- Method security business rule'u dayatır
- DB query data-level isolation uygular

Bir katman atlanırsa diğeri yakalar. **Banking security'de redundancy bug değil, feature.**

### 12. CSRF, CORS, secure headers

**CSRF (Cross-Site Request Forgery):**

- Cookie tabanlı session'lı uygulamada zorunlu (browser otomatik cookie gönderir)
- REST API'de **JWT bearer token kullanıyorsan CSRF disabled** olabilir (browser otomatik Authorization header eklemez)

```java
// REST API + JWT — CSRF disable
http.csrf(AbstractHttpConfigurer::disable);

// Web app + session cookie — CSRF enable (default)
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
);
```

**CORS:**

```java
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));

@Bean
CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.mavibank.com"));  // ASLA "*" production
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type", "Idempotency-Key"));
    config.setAllowCredentials(true);
    
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

**Secure headers:**

```java
http.headers(headers -> headers
    .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
    .frameOptions(frame -> frame.deny())   // clickjacking
    .httpStrictTransportSecurity(hsts -> hsts.maxAgeInSeconds(31536000).includeSubDomains(true))
    .referrerPolicy(ref -> ref.policy(ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
);
```

### 13. Multiple SecurityFilterChain — REST API + Admin panel

Banking uygulamasında genellikle:

- **Customer REST API** (`/v1/**`) — JWT bearer token
- **Admin web UI** (`/admin/**`) — form login + session
- **Actuator** (`/actuator/**`) — basic auth, kısıtlı IP

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    @Order(1)
    SecurityFilterChain apiSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/v1/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/v1/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> jwt
                .jwtAuthenticationConverter(jwtAuthenticationConverter())
            ))
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
        return http.build();
    }
    
    @Bean
    @Order(2)
    SecurityFilterChain actuatorSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/actuator/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().hasRole("ACTUATOR_ADMIN")
            )
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
    
    @Bean
    @Order(3)
    SecurityFilterChain webSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/admin/**")
            .authorizeHttpRequests(auth -> auth
                .anyRequest().hasRole("ADMIN")
            )
            .formLogin(Customizer.withDefaults())
            .logout(Customizer.withDefaults());
        return http.build();
    }
}
```

`@Order` önemli — daha düşük number önce eşleşir. `securityMatcher` ile hangi chain hangi URL pattern'i savunur belirlenir.

### 14. Anti-pattern'ler

**Anti-pattern 1: "Frontend yetki kontrolü yeterli"**

```javascript
// ❌ Frontend "admin değilse butonu gizle"
if (user.role === 'ADMIN') {
    showDeleteButton();
}
```

Saldırgan POSTMAN ile direkt API'ye istek atar. **Backend'de yetki kontrolü ZORUNLU**. Frontend sadece UX için, security için **DEĞİL**.

**Anti-pattern 2: "Controller'da SecurityContextHolder kullanmak"**

```java
// ❌ Service'de
@Service
class AccountService {
    public Account getById(AccountId id) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        String username = auth.getName();
        // ...
    }
}
```

Sorunlar: Test edilmesi zor (mock SecurityContextHolder gerekir), tight coupling, thread-local risk'i.

**Çözüm:** Authentication'ı parametre olarak inject et:

```java
@Service
class AccountService {
    
    @PreAuthorize("isAuthenticated()")
    public Account getById(AccountId id, @AuthenticationPrincipal Jwt jwt) {
        String customerId = jwt.getSubject();
        // ...
    }
}
```

**Anti-pattern 3: "Sadece role bazlı yetki"**

Yetki sadece role değil, **context** ile değişir. Bir teller şubesindeki müşteri için işlem yapabilir, başka şubenin müşterisi için yapamayabilir. `@PreAuthorize("hasRole('TELLER')")` yetersiz; `@PreAuthorize("@checker.canHandle(authentication, #branchId)")` doğru.

**Anti-pattern 4: "CSRF'i çıkar, sonra unut"**

REST API'de JWT kullanıyorsan CSRF gerçekten gerekmez. Ama session-based bir endpoint eklersen ve CSRF disable kalırsa **vulnerable** olursun. Stratejik karar ver, dokümante et.

**Anti-pattern 5: "Permit all and authenticate later"**

```java
// ❌
http.authorizeHttpRequests(auth -> auth
    .anyRequest().permitAll()
);
// Sonra her controller'da @PreAuthorize ekle
```

Bir method'a `@PreAuthorize` eklemeyi unutursan **public oluyor**. Default DENY:

```java
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/v1/public/**").permitAll()
    .anyRequest().authenticated()  // ← DEFAULT
);
```

**Anti-pattern 6: "Custom auth filter ile yetki kontrolü"**

Yeni başlayan dev'in tuzağı: `OncePerRequestFilter` extend et, request'i parse et, kendi yetkilendirmeni yaz. Spring Security'nin DSL'i çok daha güçlü — kendi tekerleğini icat etme.

**Anti-pattern 7: "AuthenticationManager bean'i yok, sonra @Autowired başarısız"**

Spring Security 6'da `AuthenticationManager` otomatik bean değil. Custom auth provider kullanıyorsan **manuel bean** tanımlamalısın.

---

## Önemli olabilecek araştırma kaynakları

- "Spring Security Reference" — official documentation (uzun ama referans)
- "Spring Security in Action" Laurentiu Spilca (Manning kitabı)
- Spring Security GitHub samples (`spring-security-samples`)
- "Servlet Security Architecture" Spring docs sayfası — filter chain'i resimli anlatır
- "Method Security" Spring Security docs
- Baeldung Spring Security tutorial serisi (kapsamlı)
- Spring Security Lambda DSL migration guide
- "Why don't we have WebSecurityConfigurerAdapter anymore" — Spring blog
- ThreadLocal vs Context Propagation (Spring 6 / Project Reactor)
- "Role hierarchies in Spring Security" — Stack Overflow + Baeldung

---

## Mini task'ler

### Task 8.1.1 — Bağımlılıkları ekle (10 dk)

`core-banking/pom.xml`'a:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
```

App'i ayağa kaldır. Beklenen davranış: **tüm endpoint'ler authenticate** ister, console'da random password görürsün. Bu **default** davranıştır.

### Task 8.1.2 — Minimal `SecurityFilterChain` yaz (30 dk)

`banking/common/config/SecurityConfig.java`:

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    
    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/v3/api-docs/**", "/swagger-ui/**", "/swagger-ui.html").permitAll()
                .anyRequest().authenticated()
            )
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .httpBasic(Customizer.withDefaults());   // GEÇİCİ — Topic 8.3'te JWT'ye dönüştüreceğiz
        return http.build();
    }
    
    @Bean
    PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }
    
    @Bean
    InMemoryUserDetailsManager userDetailsService(PasswordEncoder encoder) {
        UserDetails customer = User.builder()
            .username("11111111110")
            .password(encoder.encode("Password123!"))
            .roles("CUSTOMER")
            .build();
        UserDetails admin = User.builder()
            .username("admin")
            .password(encoder.encode("AdminPass1!"))
            .roles("ADMIN")
            .build();
        return new InMemoryUserDetailsManager(customer, admin);
    }
}
```

**Test et:**
- `/actuator/health` — auth'suz çalışır
- `/v1/accounts/...` — 401 (auth header yok)
- Basic auth ile (`Authorization: Basic ...`) çalışır

### Task 8.1.3 — `BankingUserDetailsService` DB tabanlı yaz (40 dk)

`InMemoryUserDetailsManager`'ı kaldır. `Customer` entity'si ekle:

```java
@Entity
@Table(name = "customers")
public class CustomerJpaEntity {
    @Id
    private UUID id;
    
    @Column(unique = true, nullable = false)
    private String tcNo;
    
    @Column(nullable = false)
    private String passwordHash;
    
    @Enumerated(EnumType.STRING)
    private CustomerSegment segment;
    
    @Column(nullable = false)
    private boolean active;
    
    @Column(nullable = false)
    private boolean locked;
    
    // ...
}
```

Migration:

```sql
CREATE TABLE customers (
    id UUID PRIMARY KEY,
    tc_no VARCHAR(11) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    segment VARCHAR(50) NOT NULL,
    active BOOLEAN NOT NULL DEFAULT TRUE,
    locked BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

`BankingUserDetailsService` yaz (üstteki "Kavramlar 6" örneği gibi).

### Task 8.1.4 — Method security ile account ownership (45 dk)

`AccountAccessChecker` bean yaz:

```java
@Component("accountAccessChecker")
public class AccountAccessChecker {
    
    private final AccountRepository accountRepository;
    
    public AccountAccessChecker(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }
    
    public boolean canAccess(String customerId, String accountId) {
        try {
            UUID ownerUuid = UUID.fromString(customerId);
            UUID accountUuid = UUID.fromString(accountId);
            return accountRepository.findById(new AccountId(accountUuid))
                .map(a -> a.getOwnerId().value().equals(ownerUuid))
                .orElse(false);
        } catch (IllegalArgumentException e) {
            return false;
        }
    }
}
```

`GetAccountService`'e `@PreAuthorize` ekle:

```java
@PreAuthorize("hasRole('ADMIN') or @accountAccessChecker.canAccess(authentication.name, #id.value().toString())")
public Account execute(AccountId id) {
    return repository.findById(id).orElseThrow(() -> new AccountNotFoundException(id));
}
```

**Test et:**
- Müşteri A kendi hesabı: 200
- Müşteri A başkasının hesabı: 403
- Admin başkasının hesabı: 200

### Task 8.1.5 — Role hierarchy ekle (20 dk)

```java
@Bean
static RoleHierarchy roleHierarchy() {
    return RoleHierarchyImpl.fromHierarchy("""
        ROLE_ADMIN > ROLE_BRANCH_MANAGER
        ROLE_BRANCH_MANAGER > ROLE_TELLER
        ROLE_TELLER > ROLE_CUSTOMER
        """);
}

@Bean
static MethodSecurityExpressionHandler methodSecurityExpressionHandler(RoleHierarchy roleHierarchy) {
    var handler = new DefaultMethodSecurityExpressionHandler();
    handler.setRoleHierarchy(roleHierarchy);
    return handler;
}
```

**Önemli:** `static` olmalı, yoksa method security bean'leri yüklenirken init order sorunu çıkar.

Test: Admin bir customer endpoint'ine ulaşabiliyor mu?

### Task 8.1.6 — Multiple `SecurityFilterChain` (30 dk)

`/v1/**` ve `/actuator/**` ve `/admin/**` için 3 ayrı chain yaz (Kavramlar 13'teki örnek). `@Order` belirle.

### Task 8.1.7 — Secure headers (15 dk)

`SecurityFilterChain`'e:

```java
.headers(headers -> headers
    .frameOptions(frame -> frame.deny())
    .contentTypeOptions(Customizer.withDefaults())
    .httpStrictTransportSecurity(hsts -> hsts
        .includeSubDomains(true)
        .maxAgeInSeconds(31536000)
    )
)
```

`/actuator/health` response header'larında `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Strict-Transport-Security: max-age=...` gör.

---

## Test yazma rehberi

### Test 8.1.1 — `@WebMvcTest` ile endpoint auth davranışı

```java
@WebMvcTest(controllers = AccountController.class)
@Import(SecurityConfig.class)
class AccountControllerSecurityTest {
    
    @Autowired MockMvc mockMvc;
    @MockBean GetAccountUseCase getAccountUseCase;
    @MockBean AccountWebMapper mapper;
    
    @Test
    void unauthenticatedRequestShouldReturn401() throws Exception {
        mockMvc.perform(get("/v1/accounts/" + UUID.randomUUID()))
            .andExpect(status().isUnauthorized());
    }
    
    @Test
    @WithMockUser(username = "11111111110", roles = "CUSTOMER")
    void authenticatedRequestShouldPassAuthFilter() throws Exception {
        UUID accountId = UUID.randomUUID();
        Account account = AccountTestBuilder.anAccount()
            .withId(accountId)
            .withOwnerId(UUID.fromString("00000000-0000-0000-0000-000000000001"))
            .build();
        when(getAccountUseCase.execute(any())).thenReturn(account);
        when(mapper.toResponse(any())).thenReturn(new AccountResponse(...));
        
        // Bu çağrı 401 olmamalı (auth geçti), 200 veya 403 olabilir
        mockMvc.perform(get("/v1/accounts/" + accountId))
            .andExpect(status().is(anyOf(200, 403)));
    }
    
    @Test
    @WithMockUser(roles = "ADMIN")
    void adminCanAccessAnyAccount() throws Exception {
        // ...
    }
}
```

### Test 8.1.2 — `@PreAuthorize` integration

```java
@SpringBootTest
@AutoConfigureMockMvc
@Testcontainers
@ActiveProfiles("test")
class AccountAccessIntegrationTest {
    
    @Container @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
    
    @Autowired MockMvc mockMvc;
    @Autowired AccountRepository accountRepository;
    
    @Test
    @WithMockUser(username = "00000000-0000-0000-0000-000000000001", roles = "CUSTOMER")
    void customerCanAccessOwnAccount() throws Exception {
        UUID ownerId = UUID.fromString("00000000-0000-0000-0000-000000000001");
        Account account = Account.open(new OwnerId(ownerId), Currency.getInstance("TRY"));
        Account saved = accountRepository.save(account);
        
        mockMvc.perform(get("/v1/accounts/" + saved.getId().value()))
            .andExpect(status().isOk());
    }
    
    @Test
    @WithMockUser(username = "00000000-0000-0000-0000-000000000999", roles = "CUSTOMER")
    void customerCannotAccessOthersAccount() throws Exception {
        UUID otherOwnerId = UUID.fromString("00000000-0000-0000-0000-000000000001");
        Account account = Account.open(new OwnerId(otherOwnerId), Currency.getInstance("TRY"));
        Account saved = accountRepository.save(account);
        
        mockMvc.perform(get("/v1/accounts/" + saved.getId().value()))
            .andExpect(status().isForbidden());
    }
    
    @Test
    @WithMockUser(roles = "ADMIN")
    void adminCanAccessAnyAccount() throws Exception {
        UUID ownerId = UUID.randomUUID();
        Account account = Account.open(new OwnerId(ownerId), Currency.getInstance("TRY"));
        Account saved = accountRepository.save(account);
        
        mockMvc.perform(get("/v1/accounts/" + saved.getId().value()))
            .andExpect(status().isOk());
    }
}
```

### Test 8.1.3 — `AccountAccessCheckerTest`

```java
@ExtendWith(MockitoExtension.class)
class AccountAccessCheckerTest {
    
    @Mock AccountRepository accountRepository;
    @InjectMocks AccountAccessChecker checker;
    
    @Test
    void shouldReturnTrueWhenCustomerOwnsAccount() {
        UUID customerId = UUID.randomUUID();
        UUID accountId = UUID.randomUUID();
        Account account = AccountTestBuilder.anAccount()
            .withId(accountId)
            .withOwnerId(customerId)
            .build();
        when(accountRepository.findById(new AccountId(accountId)))
            .thenReturn(Optional.of(account));
        
        boolean result = checker.canAccess(customerId.toString(), accountId.toString());
        
        assertThat(result).isTrue();
    }
    
    @Test
    void shouldReturnFalseWhenAccountNotFound() {
        when(accountRepository.findById(any())).thenReturn(Optional.empty());
        boolean result = checker.canAccess(UUID.randomUUID().toString(), UUID.randomUUID().toString());
        assertThat(result).isFalse();
    }
    
    @Test
    void shouldReturnFalseWhenInvalidUuid() {
        boolean result = checker.canAccess("not-a-uuid", "also-not-a-uuid");
        assertThat(result).isFalse();
    }
    
    @Test
    void shouldReturnFalseWhenDifferentOwner() {
        UUID accountId = UUID.randomUUID();
        Account account = AccountTestBuilder.anAccount()
            .withId(accountId)
            .withOwnerId(UUID.randomUUID())   // different owner
            .build();
        when(accountRepository.findById(any())).thenReturn(Optional.of(account));
        
        boolean result = checker.canAccess(UUID.randomUUID().toString(), accountId.toString());
        
        assertThat(result).isFalse();
    }
}
```

### Test 8.1.4 — Filter chain davranışı

```java
@SpringBootTest
class SecurityFilterChainTest {
    
    @Autowired ApplicationContext context;
    
    @Test
    void shouldHaveSecurityFilterChainBean() {
        Map<String, SecurityFilterChain> beans = context.getBeansOfType(SecurityFilterChain.class);
        assertThat(beans).isNotEmpty();
    }
    
    @Test
    void shouldHavePasswordEncoderBean() {
        PasswordEncoder encoder = context.getBean(PasswordEncoder.class);
        assertThat(encoder).isInstanceOf(DelegatingPasswordEncoder.class);
    }
    
    @Test
    void shouldHaveUserDetailsServiceBean() {
        UserDetailsService uds = context.getBean(UserDetailsService.class);
        assertThat(uds).isNotNull();
    }
}
```

---

## Claude-verify prompt

```
Aşağıdaki Spring Security 6 yapılandırmamı banking-grade kriterlere göre değerlendir.
Sadece eksikleri ve yanlışları işaretle, kod yazma:

1. Modern Spring Security 6 syntax:
   - `WebSecurityConfigurerAdapter` extend ediyor mu? (Olmamalı - deprecated)
   - `SecurityFilterChain` bean tabanlı mı?
   - Lambda DSL kullanılıyor mu (auth -> auth.requestMatchers(...))?
   - `antMatchers` yerine `requestMatchers` mı?

2. Authentication zinciri:
   - `UserDetailsService` implementasyonu DB tabanlı mı?
   - `loadUserByUsername` exception handling doğru mu (UsernameNotFoundException)?
   - Account locked/disabled/credentials expired durumları handle ediliyor mu?
   - PasswordEncoder bean'i `DelegatingPasswordEncoder` mu (single algoritma değil)?

3. Authorization stratejisi:
   - Default `anyRequest().authenticated()` mu (permitAll değil)?
   - Public endpoint'ler explicit (actuator/health, swagger) belirtilmiş mi?
   - Method security `@EnableMethodSecurity` ile aktif mi?
   - `@PreAuthorize` kullanılıyor mu (deprecated `@Secured` değil)?

4. Banking ownership pattern:
   - Account erişim için SpEL bean (`@accountAccessChecker`) yazılmış mı?
   - `@PreAuthorize` ile method-level enforcement var mı?
   - DB query'sinde de owner filter (defense-in-depth) var mı?
   - Admin override (`hasRole('ADMIN') or ...`) yazılmış mı?

5. Role hierarchy:
   - `RoleHierarchy` bean tanımlanmış mı?
   - `MethodSecurityExpressionHandler` ile entegre edilmiş mi?
   - Banking role yapısı (ADMIN > BRANCH_MANAGER > TELLER > CUSTOMER) anlamlı mı?
   - Bean'ler `static` mi (init order için)?

6. SecurityContext kullanımı:
   - `SecurityContextHolder.getContext()` direkt kullanımı var mı? (Olmamalı, parametre injection tercih)
   - `@AuthenticationPrincipal` ile principal inject ediliyor mu?
   - ThreadLocal leak riski (e.g. @Async ile) farkındalığı var mı?

7. Multiple filter chain:
   - REST API, Actuator, Admin için ayrı chain'ler var mı?
   - `@Order` belirlenmiş mi?
   - `securityMatcher` ile chain selection yapılmış mı?

8. Session yönetimi:
   - REST API'de `SessionCreationPolicy.STATELESS` mi?
   - CSRF gerekiyor mu, gerekmiyor mu doğru karar verilmiş mi?

9. Secure headers:
   - HSTS, frameOptions DENY, contentTypeOptions ayarlanmış mı?
   - CORS config'i `*` değil, explicit allowed origin mi?

10. Anti-pattern kontrolü:
    - Frontend'e güvenip backend yetki kontrolü atlama YOK mu?
    - Custom filter ile yetki kontrolü (DSL atlama) YOK mu?
    - Default permitAll YOK mu?

11. Test:
    - `@WithMockUser` ile auth test edilmiş mi?
    - `@WebMvcTest` ile filter chain test'i yazılmış mı?
    - Account ownership için (own / other / admin) 3 senaryo test edilmiş mi?

Her madde için PASS / FAIL / EKSIK işaretle, kanıt göster (dosya yolu/method ismi).
```

---

## Tamamlama kriterleri

- [ ] `SecurityConfig` class'ı `SecurityFilterChain` bean tabanlı yazıldı
- [ ] `@EnableWebSecurity` ve `@EnableMethodSecurity` annotation'ları var
- [ ] `BankingUserDetailsService` DB tabanlı, `customers` tablosundan çekiyor
- [ ] `PasswordEncoder` = `DelegatingPasswordEncoder`
- [ ] `AccountAccessChecker` SpEL bean yazıldı, `@PreAuthorize` ile kullanılıyor
- [ ] `RoleHierarchy` bean tanımlı (admin > teller > customer)
- [ ] Multiple `SecurityFilterChain` (en az 2: api + actuator) yazıldı
- [ ] Secure header'lar (HSTS, frameOptions DENY) ayarlandı
- [ ] CSRF disable durumu doğru karar (REST API + JWT = disable; web + session = enable)
- [ ] `SessionCreationPolicy.STATELESS` REST API için
- [ ] Test'ler `@WithMockUser` ile yazıldı
- [ ] 3-katmanlı defense (URL filter + method security + DB query) anladım ve uyguladım
- [ ] "Müşteri başkasının hesabını göremez" testi yazıldı ve geçiyor
- [ ] Admin override test'i yazıldı ve geçiyor

---

## Defter notları

1. "Spring Security 6'da `SecurityFilterChain` bean'i `WebSecurityConfigurerAdapter`'ı neden replace etti: ____."
2. "Bir HTTP request controller'a ulaşmadan önce geçtiği başlıca filter'lar (3 tanesini say): ____."
3. "`SecurityContext` neden ThreadLocal, ne tuzakları var: ____."
4. "Authentication zinciri: AuthenticationManager → AuthenticationProvider → UserDetailsService. Her birinin sorumluluğu: ____."
5. "`@PreAuthorize` vs `@PostAuthorize` farkı, banking'de hangisi tercih edilir neden: ____."
6. "`@PostFilter` neden performans tuzağı, alternatifi: ____."
7. "SpEL ifadesi içinde `@beanName.method(...)` nasıl çalışır: ____."
8. "Defense-in-depth 3 katman (URL filter, method security, DB query) — her biri neyi savunur: ____."
9. "Role hierarchy ne sağlar, banking'de örnek: ____."
10. "REST API'de CSRF neden disable edilir, ne zaman enable bırakılır: ____."
